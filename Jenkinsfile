pipeline {
    agent any
    triggers {
        // Vérifie les changements Git toutes les 5 minutes
        pollSCM('H/5 * * * *')
    }
    environment {
        DOCKER_REGISTRY = "docker.io"
        DOCKER_USERNAME = "yvesmayombo"
        DOCKER_IMAGE = "yvesmayombo/opengrc"
        KUBE_NAMESPACE = "opengrc"
        APP_NAME = "opengrc"
        APP_PORT = "80"
        NODE_PORT = "30080"
    }
    stages {
        stage('Check Git Changes') {
            steps {
                script {
                    echo "🔄 Vérification des changements Git..."
                    
                    // Vérifier que le Dockerfile existe
                    if (!fileExists('Dockerfile')) {
                        error "❌ Dockerfile non trouvé! Le pipeline s'arrête."
                    }
                    
                    // Récupérer les infos du commit
                    def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    def commitAuthor = sh(script: 'git log -1 --pretty=format:"%an"', returnStdout: true).trim()
                    def commitMessage = sh(script: 'git log -1 --pretty=format:"%s"', returnStdout: true).trim()
                    
                    env.DOCKER_TAG = "build-${BUILD_NUMBER}-${commitHash}"
                    
                    echo "📝 Dernier commit:"
                    echo "   Auteur: ${commitAuthor}"
                    echo "   Message: ${commitMessage}" 
                    echo "   Hash: ${commitHash}"
                    echo "   Tag: ${env.DOCKER_TAG}"
                    echo "   Dockerfile: ✅ Présent"
                    
                    // Afficher le contenu du Dockerfile pour vérification
                    sh 'echo "🐳 Contenu du Dockerfile:" && cat Dockerfile'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo "🐳 Construction de l'image Docker depuis le Dockerfile existant..."
                    
                    // Build avec le Dockerfile existant
                    sh "docker build -t ${DOCKER_IMAGE}:${env.DOCKER_TAG} ."
                    sh "docker images | grep ${DOCKER_IMAGE}"
                    
                    // Vérifier la taille de l'image
                    def imageSize = sh(script: "docker images ${DOCKER_IMAGE}:${env.DOCKER_TAG} --format '{{.Size}}'", returnStdout: true).trim()
                    echo "📦 Taille de l'image: ${imageSize}"
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo "📤 Pushing vers Docker Hub..."
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}:${env.DOCKER_TAG}"
                        
                        // Tag aussi comme latest si c'est la branche main
                        if (env.BRANCH_NAME == 'main') {
                            echo "🏷️ Tagging comme latest (branche main)"
                            sh "docker tag ${DOCKER_IMAGE}:${env.DOCKER_TAG} ${DOCKER_IMAGE}:latest"
                            sh "docker push ${DOCKER_IMAGE}:latest"
                        }
                    }
                    echo "✅ Image poussée avec succès: ${DOCKER_IMAGE}:${env.DOCKER_TAG}"
                }
            }
        }
        
        stage('Generate K8s Manifests') {
            steps {
                script {
                    echo "📄 Génération des manifests Kubernetes..."
                    sh 'mkdir -p k8s-auto'
                    
                    // 1. Persistent Volume
                    writeFile file: 'k8s-auto/pv.yaml', text: """
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ${APP_NAME}-pv
  labels:
    type: local
    app: ${APP_NAME}
spec:
  storageClassName: manual
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/data/${APP_NAME}"
  persistentVolumeReclaimPolicy: Retain
"""
                    
                    // 2. Persistent Volume Claim
                    writeFile file: 'k8s-auto/pvc.yaml', text: """
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ${APP_NAME}-pvc
  namespace: ${KUBE_NAMESPACE}
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
"""
                    
                    // 3. Deployment avec la nouvelle image
                    writeFile file: 'k8s-auto/deployment.yaml', text: """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${KUBE_NAMESPACE}
  labels:
    app: ${APP_NAME}
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
        imagePullPolicy: Always
        ports:
        - containerPort: ${APP_PORT}
        volumeMounts:
        - name: storage
          mountPath: /app/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: ${APP_PORT}
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: ${APP_PORT}
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: ${APP_NAME}-pvc
"""
                    
                    // 4. Service NodePort
                    writeFile file: 'k8s-auto/service.yaml', text: """
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  namespace: ${KUBE_NAMESPACE}
  labels:
    app: ${APP_NAME}
spec:
  type: NodePort
  selector:
    app: ${APP_NAME}
  ports:
    - port: ${APP_PORT}
      targetPort: ${APP_PORT}
      nodePort: ${NODE_PORT}
"""
                    echo "✅ Manifests générés avec succès"
                    sh 'ls -la k8s-auto/'
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    echo "🚀 Déploiement sur Kubernetes..."
                    
                    // Créer le namespace s'il n'existe pas
                    sh """
                    kubectl create namespace ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f - || true
                    """
                    
                    // Appliquer les manifests
                    sh "kubectl apply -f k8s-auto/ -n ${KUBE_NAMESPACE}"
                    
                    // Attendre que le déploiement soit ready
                    sh "kubectl rollout status deployment/${APP_NAME} -n ${KUBE_NAMESPACE} --timeout=300s"
                    
                    echo "✅ Déploiement terminé avec succès"
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo "🔍 Vérification de la santé de l'application..."
                    
                    // Attendre que les pods soient vraiment prêts
                    sleep 10
                    
                    sh """
                    echo "=== ÉTAT DES PODS ==="
                    kubectl get pods -n ${KUBE_NAMESPACE} -o wide
                    
                    echo "=== ÉTAT DES SERVICES ==="
                    kubectl get svc -n ${KUBE_NAMESPACE}
                    
                    echo "=== VÉRIFICATION DES LOGS ==="
                    kubectl logs -n ${KUBE_NAMESPACE} -l app=${APP_NAME} --tail=5
                    """
                    
                    // Test de connexion à l'application
                    sh """
                    echo "🧪 Test de connexion à l'application..."
                    NODE_IP=\$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                    echo "📡 Test sur: http://\${NODE_IP}:${NODE_PORT}"
                    
                    # Test avec timeout
                    timeout 30s bash -c "until curl -f http://\${NODE_IP}:${NODE_PORT} > /dev/null 2>&1; do sleep 2; done" && \\
                    echo "✅ Application accessible et responsive" || \\
                    echo "⚠️  Application déployée mais test de connexion échoué"
                    """
                }
            }
        }
    }
    
    post {
        always {
            echo "🏁 Pipeline execution terminée - Build: ${BUILD_NUMBER}"
            sh 'docker logout || true'
            
            // Nettoyage des images locales
            sh """
            docker rmi ${DOCKER_IMAGE}:${env.DOCKER_TAG} || true
            docker rmi \$(docker images -f "dangling=true" -q) || true
            """
        }
        success {
            script {
                def nodeIP = sh(script: "kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type==\"InternalIP\")].address}'", returnStdout: true).trim()
                def podStatus = sh(script: "kubectl get pods -n ${KUBE_NAMESPACE} -l app=${APP_NAME} -o jsonpath='{.items[*].status.phase}'", returnStdout: true).trim()
                
                echo """
                🎉 DÉPLOIEMENT AUTOMATIQUE RÉUSSI !
                
                📊 Résumé:
                - Image: ${DOCKER_IMAGE}:${env.DOCKER_TAG}
                - Commit: ${sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()}
                - Namespace: ${KUBE_NAMESPACE}
                - Statut Pods: ${podStatus}
                
                🌐 VOTRE APPLICATION EST ACCESSIBLE SUR:
                URL: http://${nodeIP}:${NODE_PORT}
                
                📋 État du cluster:
                """
                sh "kubectl get all -n ${KUBE_NAMESPACE}"
            }
        }
        failure {
            echo "❌ Le pipeline a échoué"
            sh """
            echo "=== DERNIERS LOGS ==="
            kubectl logs -n ${KUBE_NAMESPACE} -l app=${APP_NAME} --tail=20 || true
            
            echo "=== ÉVÉNEMENTS RÉCENTS ==="
            kubectl get events -n ${KUBE_NAMESPACE} --sort-by=.lastTimestamp | tail -10 || true
            
            echo "=== DESCRIPTION DES PODS ==="
            kubectl describe pods -n ${KUBE_NAMESPACE} -l app=${APP_NAME} || true
            """
        }
    }
}
