pipeline {
    agent {
        kubernetes {
            cloud 'kubernetes'
            label 'docker-builder'
            yaml """
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: docker
    image: docker:latest
    command: ['cat']
    tty: true
    volumeMounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
  - name: kubectl
    image: bitnami/kubectl:latest
    command: ['cat']
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
"""
        }
    }
    environment {
        DOCKER_REGISTRY = "docker.io"
        DOCKER_USERNAME = "yvesmayombo"
        DOCKER_IMAGE = "yvesmayombo/opengrc"
        KUBE_NAMESPACE = "opengrc"
        APP_NAME = "opengrc"
        APP_PORT = "8080"  // Votre app écoute sur 8080 selon le Dockerfile
        NODE_PORT = "30080"
    }
    stages {
        stage('Check Git Changes') {
            steps {
                container('docker') {
                    script {
                        echo "🔄 Vérification des changements Git..."
                        
                        if (!fileExists('Dockerfile')) {
                            error "❌ Dockerfile non trouvé!"
                        }
                        
                        def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        env.DOCKER_TAG = "build-${BUILD_NUMBER}-${commitHash}"
                        
                        echo "📝 Commit: ${commitHash}"
                        echo "🐳 Dockerfile: ✅ Présent"
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                container('docker') {
                    script {
                        echo "🐳 Construction de l'image Docker..."
                        sh "docker build -t ${DOCKER_IMAGE}:${env.DOCKER_TAG} ."
                        sh "docker images | grep ${DOCKER_IMAGE}"
                    }
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                container('docker') {
                    script {
                        echo "📤 Pushing vers Docker Hub..."
                        withCredentials([usernamePassword(
                            credentialsId: 'docker-hub-creds',
                            usernameVariable: 'DOCKER_USER',
                            passwordVariable: 'DOCKER_PASS'
                        )]) {
                            sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                            sh "docker push ${DOCKER_IMAGE}:${env.DOCKER_TAG}"
                        }
                    }
                }
            }
        }
        
        stage('Generate K8s Manifests') {
            steps {
                container('kubectl') {
                    script {
                        echo "📄 Génération des manifests Kubernetes..."
                        sh 'mkdir -p k8s-auto'
                        
                        writeFile file: 'k8s-auto/deployment.yaml', text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${KUBE_NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${DOCKER_IMAGE}:${env.DOCKER_TAG}
        ports:
        - containerPort: ${APP_PORT}
        env:
        - name: APP_ENV
          value: "production"
"""
                        
                        writeFile file: 'k8s-auto/service.yaml', text: """
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  namespace: ${KUBE_NAMESPACE}
spec:
  type: NodePort
  selector:
    app: ${APP_NAME}
  ports:
    - port: ${APP_PORT}
      targetPort: ${APP_PORT}
      nodePort: ${NODE_PORT}
"""
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    script {
                        echo "🚀 Déploiement sur Kubernetes..."
                        sh "kubectl create namespace ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f - || true"
                        sh "kubectl apply -f k8s-auto/ -n ${KUBE_NAMESPACE}"
                        sh "kubectl rollout status deployment/${APP_NAME} -n ${KUBE_NAMESPACE} --timeout=300s"
                    }
                }
            }
        }
    }
}
