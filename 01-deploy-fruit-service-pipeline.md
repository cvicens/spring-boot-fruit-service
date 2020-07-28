# Deploying the Spring Fruit Service in several ways

## SETTING THE ENVIRONMENT

```sh
export DEV_PROJECT=fruits-dev
export TEST_PROJECT=fruits-test
```

## CLEAN BEFORE RUNNING THE DEMO

```sh
oc delete project ${DEV_PROJECT}
oc delete project ${TEST_PROJECT}
```

## [OPTIONAL] ADDITIONAL DEPLOYMENT TYPES

### DEPLOY FROM GIT REPO

1. Log in as 'developer' in OCP web console.
2. Change to DEVELOPER view and create project ${DEV_PROJECT}
3. Add -> Database
3.1 Uncheck Type->Operator Backed
3.2 PostgreSQL (Non-ephemeral) then click on Instantiate Template
3.3 Database Service Name: my-database
    PostgreSQL Connection Username: luke
    PostgreSQL Connection Password: secret
    PostgreSQL Database Name: my_data
    CLICK on Create
3.4 Add -> From Git
    Git Repo URL: https://github.com/cvicens/spring-boot-fruit-service
    Java 8
    General Application: fruit-service-app 
    Name: fruit-service-git <===
    Check Deployment <===
    Click on Deployment to add env variables
    - DB_USERNAME from secret my-database...
    - DB_PASSWORD from secret my-database...
    - JAVA_OPTIONS: -Dspring.profiles.active=openshift
    Click on labels
    - app=fruit-service-git version=1.0.0

### DECORATE DATABASE AND GIT DEPLOYMENT

oc label deployment/fruit-service-git app.openshift.io/runtime=spring --overwrite=true -n ${DEV_PROJECT} && \
oc annotate deployment/fruit-service-git app.openshift.io/connects-to=my-database --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${DEV_PROJECT}

### DEPLOY JENKINS HERE TO SAVE TIME... 

Details bellow in section **"DEPLOY JENKINS"** or do as follows:

1. Log in as 'developer' in OCP web console.
2. Change to DEVELOPER view and create project ${DEV_PROJECT}
3. Add -> From Catalog
4. CICD -> Jenkins then click on Instantiate Template
   Memory Limit: 3Gi
   Disable memory intensive administrative monitors: true
   Allows use of Jenkins Update Center repository with invalid SSL certificate: true

### DEPLOY WITH F8

> CHECK JAVA VERSION (it should be 8): java -version

```sh
oc project ${DEV_PROJECT}
mvn clean fabric8:deploy -DskipTests -Popenshift
oc label dc/fruit-service-dev app.kubernetes.io/part-of=fruit-service-app --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/fruit-service-dev app.openshift.io/runtime=spring --overwrite=true -n ${DEV_PROJECT} && \
oc annotate dc/fruit-service-dev app.openshift.io/connects-to=my-database --overwrite=true -n ${DEV_PROJECT} 
```

## COMPLETE PIPELINE

### CREATE PROJECTS

> You should have created ${DEV_PROJECT} manually before... no problem with the expected error

```sh
oc new-project ${TEST_PROJECT}
oc new-project ${DEV_PROJECT}
```

### DEPLOY JENKINS

```sh
oc new-app jenkins-ephemeral -p MEMORY_LIMIT=3Gi -p JENKINS_IMAGE_STREAM_TAG=jenkins:2 -n ${DEV_PROJECT}
oc label dc/jenkins app.openshift.io/runtime=jenkins --overwrite=true -n ${DEV_PROJECT} 
```

### DATABASES [SKIP FOR DEV_PROJECT IF DONE BEFORE]

Deploy DB in ${DEV_PROJECT}

```sh
oc new-app -e POSTGRESQL_USER=luke -ePOSTGRESQL_PASSWORD=secret -ePOSTGRESQL_DATABASE=my_data centos/postgresql-10-centos7 --name=my-database -n ${DEV_PROJECT}
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app -n ${DEV_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${DEV_PROJECT} 
```

Deploy DB in ${TEST_PROJECT}

```sh
oc new-app -e POSTGRESQL_USER=luke -ePOSTGRESQL_PASSWORD=secret -ePOSTGRESQL_DATABASE=my_data centos/postgresql-10-centos7 --name=my-database -n ${TEST_PROJECT}
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app -n ${TEST_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${TEST_PROJECT} 
```

### SECURITY/ROLES

```sh
oc policy add-role-to-user edit system:serviceaccount:${DEV_PROJECT}:jenkins -n ${TEST_PROJECT} && \
oc policy add-role-to-user view system:serviceaccount:${DEV_PROJECT}:jenkins -n ${TEST_PROJECT} && \
oc policy add-role-to-user system:image-puller system:serviceaccount:${TEST_PROJECT}:default -n ${DEV_PROJECT}
```

### CREATE PIPELINE

```sh
oc apply -n ${DEV_PROJECT} -f jenkins-pipeline-complex.yaml
```

### START PIPELINE IF NO PROXY

```sh
oc start-build bc/fruit-service-pipeline-complex --env=DEV_PROJECT_NAME=${DEV_PROJECT} --env=TEST_PROJECT_NAME=${TEST_PROJECT} -n ${DEV_PROJECT}
```

### START PIPELINE IFPROXY

```sh
export HTTP_PROXY_HOST=10.2.0.40
export HTTP_PROXY_PORT=3128
export HTTPS_PROXY_HOST=10.2.0.40
export HTTPS_PROXY_PORT=3128

export KUBERNETES_HOST=172.30.0.1

export NO_PROXY="${KUBERNETES_HOST},.cluster.local,.svc,10.0.0.0/16,10.128.0.0/14,10.2.10.0/28,127.0.0.1,172.30.0.0/16,api-int.ocp4.dcst.cartasi.local,dcst.cartasi.local,etcd-0.ocp4.dcst.cartasi.local,etcd-1.ocp4.dcst.cartasi.local,etcd-2.ocp4.dcst.cartasi.local,localhost"

export NON_PROXY_HOSTS=$(echo ${NO_PROXY} | sed -e 's/,/|/g')

export MAVEN_OPTS_BASE="-Dsun.zip.disableMemoryMapping=true -Xms20m -Djava.security.egd=file:/dev/./urandom -XX:+UnlockExperimentalVMOptions -Dsun.zip.disableMemoryMapping=true"

export MAVEN_OPTS="${MAVEN_OPTS_BASE} -Dhttp.proxyHost=${HTTP_PROXY_HOST} -Dhttp.proxyPort=${HTTP_PROXY_PORT} -Dhttps.proxyHost=${HTTPS_PROXY_HOST} -Dhttps.proxyPort=${HTTPS_PROXY_PORT} -Dhttp.nonProxyHosts=\"${NON_PROXY_HOSTS}\""

oc start-build bc/fruit-service-pipeline-complex \
  --env=DEV_PROJECT_NAME=${DEV_PROJECT} --env=TEST_PROJECT_NAME=${TEST_PROJECT} \
  --env=HTTP_PROXY="http://${HTTP_PROXY_HOST}:${HTTP_PROXY_PORT}" \
  --env=HTTPS_PROXY="http://${HTTPS_PROXY_HOST}:${HTTPS_PROXY_PORT}" \
  --env=NO_PROXY="${NO_PROXY}" \
  --env=MAVEN_OPTS="${MAVEN_OPTS}" \
  -n ${DEV_PROJECT}
```





# Troubleshooting Pipelines

oc import-image jenkins-alt:4.3.26 --from=registry.redhat.io/openshift4/ose-jenkins:v4.3.26 --confirm --scheduled=true -n openshift

jenkins-alt:4.3.26

```sh
oc import-image jenkins-alt:4.3.26 --from=registry.redhat.io/openshift4/ose-jenkins:v4.3.26 --confirm --scheduled=true -n openshift
oc new-app jenkins-ephemeral -p MEMORY_LIMIT=3Gi -p JENKINS_IMAGE_STREAM_TAG=jenkins-alt:4.3.26 -n ${DEV_PROJECT}
```



