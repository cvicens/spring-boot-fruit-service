# Setting up the enviroment

## Requirements 

* Operators Installed: Elastic Search, Kiali, Red Hat Jaeger, Red Hat Service Mesh 
* Service Mesh Control Plane deployed

```sh
export DEV_PROJECT=fruit-service-dev
export TEST_PROJECT=fruit-service-test
export SMCP_PROJECT=fruit-smcp
export VIRTUAL_SERVICE_NAME=fruit-service-git
export DESTINATION_RULE_NAME=fruit-service-git
```

## Create Service Mesh Control Plane

```sh
oc new-project ${SMCP_PROJECT}

cat << EOF | oc -n ${SMCP_PROJECT} apply -f -
apiVersion: maistra.io/v1
kind: ServiceMeshControlPlane
metadata:
  name: basic-install
spec:
  version: v1.1
  istio:
    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
      istio-ingressgateway:
        autoscaleEnabled: false
        ior_enabled: false
    mixer:
      policy:
        autoscaleEnabled: false
      telemetry:
        autoscaleEnabled: false
    pilot:
      autoscaleEnabled: false
      traceSampling: 100
    kiali:
      enabled: true
    grafana:
      enabled: true
    tracing:
      enabled: true
      jaeger:
        template: all-in-one
EOF
```

Check estatus until you get STATUS === UpdateSuccessful

> OUTPUT: 
>
> ```
> NAME            READY   STATUS             TEMPLATE   VERSION   AGE
> basic-install   9/9     UpdateSuccessful   default    v1.1      7d17h
> ```

```sh
oc get smcp -n ${SMCP_PROJECT} -w
```

## Add ${TEST_PROJECT} to the ServiceMeshMemberRoll

Edit the **ServiceMeshMemberRoll** in project **${SMCP_PROJECT}**

```sh
cat << EOF | oc -n ${SMCP_PROJECT} apply -f -
apiVersion: maistra.io/v1
kind: ServiceMeshMemberRoll
metadata:
  name: default
  namespace: ${SMCP_PROJECT}
spec:
  members:
    - ${DEV_PROJECT}
    - ${TEST_PROJECT}
EOF
```

## Enabling automatic sidecar injection

Red Hat OpenShift Service Mesh relies on a proxy sidecar within the application’s pod to provide Service Mesh capabilities to the application. You can enable automatic sidecar injection or manage it manually. Red Hat recommends automatic injection using the annotation with no need to label projects. This ensures that your application contains the appropriate configuration for the Service Mesh upon deployment. This method requires fewer privileges and does not conflict with other OpenShift capabilities such as builder pods.

> Note: The upstream version of Istio injects the sidecar by default if you have labeled the project. Red Hat OpenShift Service Mesh requires you to opt in to having the sidecar automatically injected to a deployment, so you are not required to label the project. This avoids injecting a sidecar if it is not wanted (for example, in build or deploy pods).

The webhook checks the configuration of pods deploying into all projects to see if they are opting in to injection with the appropriate annotation.


## Add sidecars
OpenShift Service Mesh requires that applications "opt-in" to being part of a service mesh by default. To "opt-in" an app, you need to add an annotation which is a flag to istio to attach a sidecar and bring the app into the mesh.

First, do the databases and wait for them to be re-deployed:

```sh
oc patch dc/my-database -n ${DEV_PROJECT} --type='json' -p '[{"op":"add","path":"/spec/template/metadata/annotations", "value": {"sidecar.istio.io/inject": "'"true"'"}}]'
oc rollout latest dc/my-database -n ${DEV_PROJECT} && \
oc rollout status -w dc/my-database -n ${DEV_PROJECT}
```

This should take about 1 minute to finish.

Next, let’s add sidecars to our services and wait for them to be re-deployed:

```sh
oc patch deployment/fruit-service-git -n ${DEV_PROJECT} --type='json' -p '[{"op":"add","path":"/spec/template/metadata/annotations", "value": {"sidecar.istio.io/inject": "'"true"'"}}]' && \
oc rollout status -w deployment/fruit-service-git -n ${DEV_PROJECT}
```

## Adapt Kubernetes Service object for Istio

Service->spec->ports->name ==> proto(-suffix) ==> http-8080

```sh
cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: v1
kind: Service
metadata:
  annotations:
    app.openshift.io/vcs-ref: master
    app.openshift.io/vcs-uri: https://github.com/cvicens/spring-boot-fruit-service
    openshift.io/generated-by: OpenShiftWebConsole
  labels:
    app: fruit-service-git
    app.kubernetes.io/component: fruit-service-git
    app.kubernetes.io/instance: fruit-service-git
    app.kubernetes.io/name: java
    app.kubernetes.io/part-of: fruit-service-app
    app.openshift.io/runtime: java
    app.openshift.io/runtime-version: "8"
    version: 1.0.0
  name: fruit-service-git
  namespace: ${DEV_PROJECT}
spec:
  ports:
  - name: http-8080
    port: 8080
    protocol: TCP
    targetPort: 8080
  - name: http-8443
    port: 8443
    protocol: TCP
    targetPort: 8443
  - name: http-8778
    port: 8778
    protocol: TCP
    targetPort: 8778
  selector:
    app: fruit-service-git
    deploymentconfig: fruit-service-git
  sessionAffinity: None
  type: ClusterIP
EOF
```

# Create basic Istio objects

## Gateway

```sh
cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: fruit-service-gateway
  namespace: ${DEV_PROJECT}
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - '*'
    port:
      name: http
      number: 80
      protocol: HTTP
EOF
```

## Virtual Service

```sh
export INGRESS_ROUTE_HOST=$(oc get route/istio-ingressgateway -o json -n $SMCP_PROJECT | jq -r '.status.ingress[0].host')

cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ${VIRTUAL_SERVICE_NAME}
  namespace: ${DEV_PROJECT}
spec:
  hosts:
  - "${INGRESS_ROUTE_HOST}"
  - fruit-service-git.${DEV_PROJECT}.svc.cluster.local
  gateways:
  - fruit-service-gateway
  http:
    - match:
        - uri:
            prefix: /api/fruits
        - uri:
            prefix: /setup
        - uri:
            exact: /
      route:
        - destination:
            host: fruit-service-git
            port:
              number: 8080
EOF
```

Let's test the virtual service we just created.

```sh
curl -vs http://${INGRESS_ROUTE_HOST}/api/fruits
```

## Break the service by adding a delay

Set delay to 2000 (2s) more than Virtual Service timeout (1s) and less than the readiness probe timeout (3s)

```sh
curl http://${INGRESS_ROUTE_HOST}/setup/delay/2000 && echo
```

Check delay

```sh
time curl -vs http://${INGRESS_ROUTE_HOST}/api/fruits
```

It should around 2+ seconds

## Add timeout to our Virtual Service

NOTE: timeout: 1.000s

```sh
cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ${VIRTUAL_SERVICE_NAME}
  namespace: ${DEV_PROJECT}
spec:
  hosts:
  - "${INGRESS_ROUTE_HOST}"
  - fruit-service-git.${DEV_PROJECT}.svc.cluster.local
  gateways:
  - fruit-service-gateway
  http:
    - match:
        - uri:
            prefix: /api/fruits
        - uri:
            prefix: /setup
        - uri:
            exact: /
      route:
        - destination:
            host: fruit-service-git
            port:
              number: 8080
      timeout: 1.000s
EOF
```

Have a look in Kiali:

```sh
export KIALI_ROUTE_HOST=$(oc get route/kiali -o json -n $SMCP_PROJECT | jq -r '.status.ingress[0].host')

echo "open https://${KIALI_ROUTE_HOST}/console/namespaces/${DEV_PROJECT}/istio/virtualservices/${VIRTUAL_SERVICE_NAME}?list=overview"
```

Check delay

```sh
time curl -vs http://${INGRESS_ROUTE_HOST}/api/fruits
```

Now timeout is 1+ seconds... but you get a 504 error.

## Fix the service by setting delay to zero

Set delay to 2000 (2s) more than Virtual Service timeout (1s) and less than the readiness probe timeout (3s)

```sh
curl http://${INGRESS_ROUTE_HOST}/setup/delay/0 && echo
```

Check delay

```sh
time curl -vs http://${INGRESS_ROUTE_HOST}/api/fruits
```

It should around 0+ seconds

## Adding a Circuit Breaker to our service

Circuit Breaking configuration is set at the DestinationRule level.

```sh
cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: ${DESTINATION_RULE_NAME}
spec:
  host: fruit-service-git
  subsets:
  - name: v1
    labels:
      app: fruit-service-git
      version: 1.0.0
    trafficPolicy:
      connectionPool:
        http:
          http1MaxPendingRequests: 2
          maxRequestsPerConnection: 10
        tcp:
          maxConnections: 10
      outlierDetection:
        consecutive5xxErrors: 2
        interval: 10s
        baseEjectionTime: 60s
        maxEjectionPercent: 100
EOF
```

Now set replicas to two... we want to set a pod in error state but the other one in normal state.

```sh
oc scale --replicas=2 deployment fruit-service-git -n ${DEV_PROJECT}
```

Let's set one of the pods in error state

```sh
curl http://${INGRESS_ROUTE_HOST}/setup/error && echo
```

## Update our Virtual Service to point to the DR subset v1

We point to a subset for which we have defined a Circuit Breaker.

```sh
cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ${VIRTUAL_SERVICE_NAME}
  namespace: ${DEV_PROJECT}
spec:
  hosts:
  - "${INGRESS_ROUTE_HOST}"
  - fruit-service-git.${DEV_PROJECT}.svc.cluster.local
  gateways:
  - fruit-service-gateway
  http:
    - match:
        - uri:
            prefix: /api/fruits
        - uri:
            prefix: /setup
        - uri:
            exact: /
      route:
        - destination:
            host: fruit-service-git
            port:
              number: 8080
            subset: v1
      timeout: 1.000s
EOF
```

Let's send request to see how we stop seeing errors after some errors.

```sh
for i in {1..180}; do time curl -s http://${INGRESS_ROUTE_HOST}/api/fruits; echo; sleep 1; done
```

Let's remove errors completely by adding one retry.

```sh
cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ${VIRTUAL_SERVICE_NAME}
  namespace: ${DEV_PROJECT}
spec:
  hosts:
  - "${INGRESS_ROUTE_HOST}"
  - fruit-service-git.${DEV_PROJECT}.svc.cluster.local
  gateways:
  - fruit-service-gateway
  http:
    - match:
        - uri:
            prefix: /api/fruits
        - uri:
            prefix: /setup
        - uri:
            exact: /
      route:
        - destination:
            host: fruit-service-git
            port:
              number: 8080
            subset: v1
      timeout: 1.000s
      retries:
        attempts: 1
        #perTryTimeout: 1s
        retryOn: 5xx
EOF
```

Let's send some more requests to see how we stop seeing any errors.

> NOTE: Clear screen!

```sh
for i in {1..180}; do time curl -s http://${INGRESS_ROUTE_HOST}/api/fruits; echo; sleep 1; done
```

