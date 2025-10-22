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
    image: alpine/k8s:1.30.0
    command: ['cat']
    tty: true
  volumes:
  - name: docker-sock
    hostPath:
      path: "/var/run/docker.sock"
"""
        }
    }
    environment {
        DOCKER_REGISTRY = "docker.io"
        DOCKER_USERNAME = "yvesmayombo"
        DOCKER_IMAGE = "yvesmayombo/opengrc"
        KUBE_NAMESPACE = "opengrc"
        APP_NAME = "opengrc"
        APP_PORT = "8080"
        NODE_PORT = "30080"
    }
    stages {
        stage('Check Git Changes') {
            steps {
                container('docker') {
                    script {
                        // RÉPARATION : Configurer Git safe.directory
                        sh 'git config --global --add safe.directory /home/jenkins/agent/workspace/opengrc'
                        
                        echo "🔄 Vérification des changements Git..."
                        echo "⏰ Poll SCM configuré: */5 * * * * (toutes les 5 minutes)"
                        echo "🚀 CI/CD AUTOMATIQUE - Déclenché par modification Jenkinsfile"
                        
                        if (!fileExists('Dockerfile')) {
                            error "❌ Dockerfile non trouvé!"
                        }
                        
                        def commitHash = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        env.DOCKER_TAG = "build-${BUILD_NUMBER}-${commitHash}"
                        
                        echo "📝 Commit: ${commitHash}"
                        echo "🐳 Dockerfile: ✅ Présent"
                        echo "✅ Modification test CI/CD - Aucun impact fonctionnel"
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
                        echo "📋 Inclut: PV, PVC, Deployment, Service NodePort"
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
                        
                        // 3. Deployment avec volume
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
        volumeMounts:
        - name: storage
          mountPath: /var/www/html/storage
        - name: storage
          mountPath: /var/www/html/bootstrap/cache
        env:
        - name: APP_ENV
          value: "production"
        - name: APP_DEBUG
          value: "false"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /
            port: ${APP_PORT}
          initialDelaySeconds: 60
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: ${APP_PORT}
          initialDelaySeconds: 30
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
                        echo "📦 Déploiement AUTOMATIQUE déclenché par CI/CD"
                        
                        // Créer le namespace
                        sh "kubectl create namespace ${KUBE_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f - || true"
                        
                        // Appliquer PV d'abord (cluster scope)
                        sh "kubectl apply -f k8s-auto/pv.yaml"
                        
                        // Puis le reste dans le namespace
                        sh "kubectl apply -f k8s-auto/pvc.yaml -n ${KUBE_NAMESPACE}"
                        sh "kubectl apply -f k8s-auto/deployment.yaml -n ${KUBE_NAMESPACE}"
                        sh "kubectl apply -f k8s-auto/service.yaml -n ${KUBE_NAMESPACE}"
                        
                        // Attendre le déploiement
                        sh "kubectl rollout status deployment/${APP_NAME} -n ${KUBE_NAMESPACE} --timeout=300s"
                        
                        echo "✅ Déploiement terminé avec succès"
                        echo "🎉 CI/CD AUTOMATIQUE FONCTIONNEL !"
                    }
                }
            }
        }
        
        stage('Verify Storage') {
            steps {
                container('kubectl') {
                    script {
                        echo "🔍 Vérification du storage..."
                        sh """
                        echo "=== PV ==="
                        kubectl get pv
                        echo "=== PVC ==="
                        kubectl get pvc -n ${KUBE_NAMESPACE}
                        echo "=== PODS ==="
                        kubectl get pods -n ${KUBE_NAMESPACE}
                        echo "=== SERVICES ==="
                        kubectl get svc -n ${KUBE_NAMESPACE}
                        """
                    }
                }
            }
        }
        
        stage('Health Check') {
            steps {
                container('kubectl') {
                    script {
                        echo "🏥 Health Check de l'application..."
                        sh """
                        # Attendre que l'application soit prête
                        sleep 30
                        
                        # Récupérer l'IP du node
                        NODE_IP=\$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
                        echo "🌐 Test de l'application sur: http://\${NODE_IP}:${NODE_PORT}"
                        
                        # Test de connexion
                        if curl -f http://\${NODE_IP}:${NODE_PORT} > /dev/null 2>&1; then
                            echo "✅ Application accessible et responsive"
                        else
                            echo "⚠️  Application déployée mais non accessible - vérifiez les logs"
                        fi
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            echo "🏁 Pipeline execution terminée - Build: ${BUILD_NUMBER}"
        }
        success {
            echo "🎉 SUCCÈS: Application déployée avec storage persistant!"
            script {
                def nodeIP = sh(script: "kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type==\"InternalIP\")].address}'", returnStdout: true).trim()
                echo "🌐 URL: http://${nodeIP}:${NODE_PORT}"
                echo "💾 Storage: PV et PVC configurés pour la persistance"
                echo "🚀 CI/CD AUTOMATIQUE: OPÉRATIONNEL !"
                
                // Afficher le résumé final
                sh """
                echo ""
                echo "📊 RÉSUMÉ DU DÉPLOIEMENT:"
                echo "=========================="
                kubectl get all -n ${KUBE_NAMESPACE}
                echo ""
                kubectl get pv,pvc -n ${KUBE_NAMESPACE}
                """
            }
        }
        failure {
            echo "❌ ÉCHEC: Vérifiez les logs"
            script {
                sh """
                echo "=== DERNIERS LOGS ==="
                kubectl logs -n ${KUBE_NAMESPACE} -l app=${APP_NAME} --tail=20 2>/dev/null || echo "Aucun pod trouvé"
                
                echo "=== ÉVÉNEMENTS RÉCENTS ==="
                kubectl get events -n ${KUBE_NAMESPACE} --sort-by=.lastTimestamp 2>/dev/null | tail -10 || echo "Aucun événement"
                
                echo "=== DESCRIPTION DES PODS ==="
                kubectl describe pods -n ${KUBE_NAMESPACE} -l app=${APP_NAME} 2>/dev/null || echo "Aucun pod à décrire"
                """
            }
        }
    }
}
