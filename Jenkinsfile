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
                script {
                    qualityGates = readYaml file: 'quality-gates.yaml'
                }
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
