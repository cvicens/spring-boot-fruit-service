## ENVIRONMENT
export DEV_PROJECT=fruit-service-dev
export TEST_PROJECT=fruit-service-test

## CLEAN BEFORE RUNNING THE DEMO
oc delete project ${DEV_PROJECT}
oc delete project ${TEST_PROJECT}

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

Details bellow in section ### DEPLOY JENKINS

### DEPLOY WITH F8

> CHECK JAVA VERSION (it should be 8): java -version

oc project ${DEV_PROJECT}
mvn clean fabric8:deploy -DskipTests -Popenshift
oc label dc/fruit-service-dev app.kubernetes.io/part-of=fruit-service-app --overwrite=true -n ${DEV_PROJECT} && \
oc label dc/fruit-service-dev app.openshift.io/runtime=spring --overwrite=true -n ${DEV_PROJECT} && \
oc annotate dc/fruit-service-dev app.openshift.io/connects-to=my-database --overwrite=true -n ${DEV_PROJECT} 

## COMPLETE PIPELINE

### PROJECTS
oc new-project ${TEST_PROJECT}
oc new-project ${DEV_PROJECT}

### DEPLOY JENKINS
#oc new-app jenkins-ephemeral -p MEMORY_LIMIT=3Gi -p JENKINS_IMAGE_STREAM_TAG=jenkins:4.2.5 -n ${DEV_PROJECT}
oc new-app jenkins-ephemeral -p MEMORY_LIMIT=3Gi -p JENKINS_IMAGE_STREAM_TAG=jenkins:2 -n ${DEV_PROJECT}
oc label dc/jenkins app.openshift.io/runtime=jenkins --overwrite=true -n ${DEV_PROJECT} 

### DATABASES [SKIP IN DEV_PROJECT IF DONE BEFORE]
oc new-app -e POSTGRESQL_USER=luke -ePOSTGRESQL_PASSWORD=secret -ePOSTGRESQL_DATABASE=my_data centos/postgresql-10-centos7 --name=my-database -n ${DEV_PROJECT}
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app -n ${DEV_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${DEV_PROJECT} 

oc new-app -e POSTGRESQL_USER=luke -ePOSTGRESQL_PASSWORD=secret -ePOSTGRESQL_DATABASE=my_data centos/postgresql-10-centos7 --name=my-database -n ${TEST_PROJECT}
oc label dc/my-database app.kubernetes.io/part-of=fruit-service-app -n ${TEST_PROJECT} && \
oc label dc/my-database app.openshift.io/runtime=postgresql --overwrite=true -n ${TEST_PROJECT} 

### SECURITY/ROLES
oc policy add-role-to-user edit system:serviceaccount:${DEV_PROJECT}:jenkins -n ${TEST_PROJECT} && \
oc policy add-role-to-user view system:serviceaccount:${DEV_PROJECT}:jenkins -n ${TEST_PROJECT} && \
oc policy add-role-to-user system:image-puller system:serviceaccount:${TEST_PROJECT}:default -n ${DEV_PROJECT}

### CREATE PIPELINE
oc apply -n ${DEV_PROJECT} -f jenkins-pipeline-complex.yaml

### START PIPELINE
oc start-build bc/fruit-service-pipeline-complex -n ${DEV_PROJECT}



###### PROXY
http_proxy=xxx.xxx.xxx.xxx:8080
HTTP_PROXY=xxx.xxx.xxx.xxx:8080
HTTPS_PROXY=xxx.xxx.xxx.xxx:8080
NO_PROXY=localhost,127.0.0.1,.svc,.cluster.local,172.30.0.1
JENKINS_JAVA_OVERRIDES='-Dhttp.proxyHost=xxx.xxx.xxx.xxx -Dhttp.proxyPort=8080 -Dhttps.proxyHost=xxx.xxx.xxx.xxx  -Dhttps.proxyPort=8080 -Dhttp.nonProxyHosts="localhost|127.*|*.svc|*.cluster.local|172.30.*"'


- name: http_proxy
  value: 'http://10.2.0.40:3128'
- name: https_proxy
  value: 'http://10.2.0.40:3128'
- name: no_proxy
  value: >-
    10.2.10.0/28,.dcst.cartasi.local,localhost,kubernetes.default,.svc.cluster.local,127.,.svc,.cluster.local,172.30.
- name: http_proxy
  value: 'http://10.2.0.40:3128'
- name: JENKINS_JAVA_OVERRIDES
  value: >-
    -Dhttp.proxyHost=10.2.0.40 -Dhttp.proxyPort=3128 -Dhttps.proxyHost=10.2.0.40 -Dhttps.proxyPort=3128 -Dhttp.nonProxyHosts='10.2.10.0/28|.dcst.cartasi.local|localhost|172.30.0.1|kubernetes.default'



===================== THE END =====================


==> Cambiar pom.xml -> JDK version

git commit -a -m "jdk 8:1.5"
git push origin master

oc start-build bc/fruit-service-pipeline-complex -n ${DEV_PROJECT}

==> Probar jdk version en DEV antes de aprobar!

==> Aprobar y probar jdk version en TEST

