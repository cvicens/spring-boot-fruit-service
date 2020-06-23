# Setting up the enviroment

## Requirements 

* Operators Installed: Elastic Search, Kiali, Red Hat Jaeger, Red Hat Service Mesh 
* Service Mesh Control Plane deployed

```sh
export DEV_PROJECT=fruit-service-dev
export TEST_PROJECT=fruit-service-test
export SMCP_PROJECT=fruit-smcp
export VIRTUAL_SERVICE_NAME=fruit-service-default
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
oc patch dc/my-database -n ${DEV_PROJECT} --type='json' -p '[{"op":"add","path":"/spec/template/metadata/annotations", "value": {"sidecar.istio.io/inject": "'"true"'"}}]' && \
oc rollout latest dc/my-database -n ${DEV_PROJECT} && \
oc rollout status -w dc/my-database -n ${DEV_PROJECT}
```

This should take about 1 minute to finish.

Next, let’s add sidecars to our services and wait for them to be re-deployed:

```sh
oc patch dc/fruit-service -n ${DEV_PROJECT} --type='json' -p '[{"op":"add","path":"/spec/template/metadata/annotations", "value": {"sidecar.istio.io/inject": "'"true"'"}}]' && \
oc rollout latest dc/fruit-service -n ${DEV_PROJECT} && \
oc rollout status -w dc/fruit-service -n ${DEV_PROJECT}
```

## Adapt Kubernetes Service object for Istio

Service->spec->ports->name ==> proto(-suffix) ==> http-8080

```sh
cat << EOF | oc -n ${DEV_PROJECT} apply -f -
apiVersion: v1
kind: Service
metadata:
  annotations:
    openshift.io/generated-by: OpenShiftNewApp
  labels:
    app: fruit-service
    app.kubernetes.io/component: fruit-service
    app.kubernetes.io/instance: fruit-service
  name: fruit-service
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
    deploymentconfig: fruit-service
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
  gateways:
  - fruit-service-gateway
  http:
    - match:
        - uri:
            exact: /api/fruits
        - uri:
            exact: /
      route:
        - destination:
            host: fruit-service
            port:
              number: 8080
EOF
```

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
  gateways:
  - fruit-service-gateway
  http:
    - match:
        - uri:
            exact: /api/fruits
        - uri:
            exact: /
      route:
        - destination:
            host: fruit-service
            port:
              number: 8080
      timeout: 1.000s
EOF
```

Have a look in Kiali:

```sh
export KIALI_ROUTE_HOST=$(oc get route/kiali -o json -n $SMCP_PROJECT | jq -r '.status.ingress[0].host')

echo https://${KIALI_ROUTE_HOST}/console/namespaces/${DEV_PROJECT}/istio/virtualservices/${VIRTUAL_SERVICE_NAME}?list=overview
```
