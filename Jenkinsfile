pipeline {
    agent {
        kubernetes {
            defaultContainer 'jdk'
            yaml '''
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: jdk
      image: docker.io/eclipse-temurin:20.0.1_9-jdk
      command:
        - cat
      tty: true
      volumeMounts:
        - name: m2-cache
          mountPath: /root/.m2
    - name: podman
      image: quay.io/containers/podman:v4.5.1
      command:
        - cat
      tty: true
      securityContext:
        runAsUser: 0
        privileged: true
    - name: kubectl
      image: docker.io/bitnami/kubectl:1.27.3
      command:
        - cat
      tty: true
      securityContext:
        runAsUser: 0
        privileged: true
  volumes:
    - name: m2-cache
      hostPath:
        path: /data/m2-cache
        type: DirectoryOrCreate
'''
        }
    }
    environment {
        APP_NAME = getPomArtifactId()
        APP_VERSION = getPomVersionNoQualifier()
        APP_CONTEXT_ROOT = '/' // it should be '/' or '<some-context>/'
        APP_LISTENING_PORT = '8080'
        APP_JACOCO_PORT = '6300'
        CONTAINER_REGISTRY_URL = 'docker.io'
        IMAGE_ORG = 'serracnt' // change it to your own organization at Docker.io!
        IMAGE_NAME = "$IMAGE_ORG/$APP_NAME"
        IMAGE_SNAPSHOT = "$IMAGE_NAME:$APP_VERSION-snapshot-$BUILD_NUMBER" // tag for snapshot version
        IMAGE_SNAPSHOT_LATEST = "$IMAGE_NAME:latest-snapshot" // tag for latest snapshot version
        IMAGE_GA = "$IMAGE_NAME:$APP_VERSION" // tag for GA version
        IMAGE_GA_LATEST = "$IMAGE_NAME:latest" // tag for latest GA version
        EPHTEST_CONTAINER_NAME = "ephtest-$APP_NAME-snapshot-$BUILD_NUMBER"
        EPHTEST_BASE_URL = "http://$EPHTEST_CONTAINER_NAME:$APP_LISTENING_PORT".concat("/$APP_CONTEXT_ROOT".replace('//', '/'))
        // credentials
        KUBERNETES_CLUSTER_CRED_ID = 'k3s-lima-vm-kubeconfig2'
        CONTAINER_REGISTRY_CRED = credentials("docker-hub-$IMAGE_ORG")
    }
    stages {
        stage('Prepare environment') {
            steps {
                echo '-=- prepare environment -=-'
                echo "APP_NAME: ${APP_NAME}\nAPP_VERSION: ${APP_VERSION}"
                echo "the name for the epheremeral test container to be created is: $EPHTEST_CONTAINER_NAME"
                echo "the base URL for the epheremeral test container is: $EPHTEST_BASE_URL"
                sh 'java -version'
                sh './mvnw --version'
                container('podman') {
                    sh 'podman --version'
                    sh "podman login $CONTAINER_REGISTRY_URL -u $CONTAINER_REGISTRY_CRED_USR -p $CONTAINER_REGISTRY_CRED_PSW"
                 }
                container('kubectl') {
                    withKubeConfig([credentialsId: "$KUBERNETES_CLUSTER_CRED_ID"]) {
                    sh 'kubectl version'
                    }
                }
            }
        }
     
        stage('Compile') {
                steps {
                    echo '-=- compiling project -=-'
                    sh './mvnw compile'
                }
        }
     
        stage('Unit tests') {
                steps {
                    echo '-=- execute unit tests -=-'
                    sh './mvnw test org.jacoco:jacoco-maven-plugin:report'
                    junit 'target/surefire-reports/*.xml'
                    jacoco execPattern: 'target/jacoco.exec'
                }
        }
     
        stage('Mutation tests') {
                steps {
                    echo '-=- execute mutation tests -=-'
                    sh './mvnw org.pitest:pitest-maven:mutationCoverage'
                }
        }
     
        stage('Package') {
                steps {
                    echo '-=- packaging project -=-'
                    sh './mvnw package -DskipTests'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
        }
     
        stage('Build & push container image') {
                steps {
                    echo '-=- build & push container image -=-'
                    container('podman') {
                        sh "podman build -t $IMAGE_SNAPSHOT ."
                        sh "podman tag $IMAGE_SNAPSHOT $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT"
                        sh "podman push $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT"
                        sh "podman tag $IMAGE_SNAPSHOT $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT_LATEST"
                        sh "podman push $CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT_LATEST"
                    }
                }
        }
        stage('Run container image') {
                steps {
                    echo '-=- run container image -=-'
                    container('kubectl') {
                        withKubeConfig([credentialsId: "$KUBERNETES_CLUSTER_CRED_ID"]) {
                            sh "kubectl run $EPHTEST_CONTAINER_NAME --image=$CONTAINER_REGISTRY_URL/$IMAGE_SNAPSHOT --env=JAVA_OPTS=-javaagent:/jacocoagent.jar=output=tcpserver,address=*,port=$APP_JACOCO_PORT --port=$APP_LISTENING_PORT"
                            sh "kubectl expose pod $EPHTEST_CONTAINER_NAME --port=$APP_LISTENING_PORT"
                            sh "kubectl expose pod $EPHTEST_CONTAINER_NAME --port=$APP_JACOCO_PORT --name=$EPHTEST_CONTAINER_NAME-jacoco"
                        }
                    }
                }
        }
     
        stage('Integration tests') {
                steps {
                    echo '-=- execute integration tests -=-'
                    sh "curl --retry 10 --retry-connrefused --connect-timeout 5 --max-time 5 ${EPHTEST_BASE_URL}actuator/health"
                    sh "./mvnw failsafe:integration-test failsafe:verify -Dtest.target.server.url=$EPHTEST_BASE_URL"
                    sh "java -jar target/dependency/jacococli.jar dump --address $EPHTEST_CONTAINER_NAME-jacoco --port $APP_JACOCO_PORT --destfile target/jacoco-it.exec"
                    sh 'mkdir target/site/jacoco-it'
                    sh 'java -jar target/dependency/jacococli.jar report target/jacoco-it.exec --classfiles target/classes --xml target/site/jacoco-it/jacoco.xml'
                    junit 'target/failsafe-reports/*.xml'
                    jacoco execPattern: 'target/jacoco-it.exec'
                }
        }
     
        stage('Performance tests') {
                steps {
                    echo '-=- execute performance tests -=-'
                    sh "curl --retry 10 --retry-connrefused --connect-timeout 5 --max-time 5 ${EPHTEST_BASE_URL}actuator/health"
                    sh "./mvnw jmeter:configure@configuration jmeter:jmeter jmeter:results -Djmeter.target.host=$EPHTEST_CONTAINER_NAME -Djmeter.target.port=$APP_LISTENING_PORT -Djmeter.target.root=$APP_CONTEXT_ROOT"
                    perfReport sourceDataFiles: 'target/jmeter/results/*.csv'
                }
        }
    }
}
def getPomVersion() {
    return readMavenPom().version
}
def getPomVersionNoQualifier() {
    return readMavenPom().version.split('-')[0]
}
def getPomArtifactId() {
    return readMavenPom().artifactId
}
