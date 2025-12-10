pipeline {
    agent any
    
    tools {
        maven 'M2_HOME'
    }
    
    environment {
        DOCKER_IMAGE = 'mayamarzouki/student-management'
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        K8S_NAMESPACE = 'devops'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/Maya-Marzouki/DevOpsPipeline-.git'
            }
        }

        stage('Setup Kubernetes') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=/var/lib/jenkins/.kube/config

                        echo "=== Configuration Kubernetes ==="

                        # Créer le namespace
                        kubectl create namespace ${env.K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

                        # Vérifier la connexion
                        kubectl cluster-info
                    """
                }
            }
        }

        stage('Build & Test') {
            steps {
                sh 'mvn clean compile test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Utiliser le SonarQube existant (pas celui sur K8S pour le moment)
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=student-management'
                }
            }
        }

        stage('Package') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker') {
            steps {
                sh """
                    docker build -t ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} .
                    docker tag ${env.DOCKER_IMAGE}:${env.DOCKER_TAG} ${env.DOCKER_IMAGE}:latest
                """
            }
        }

        stage('Push Docker') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh """
                        echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                        docker push ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
                        docker push ${env.DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Deploy SonarQube on K8S') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=/var/lib/jenkins/.kube/config

                        echo "=== Déploiement de SonarQube sur K8S ==="

                        # Déployer SonarQube
                        kubectl apply -f sonarqube-persistentvolume.yaml -n ${env.K8S_NAMESPACE} 2>/dev/null || echo "PV déjà existant"
                        kubectl apply -f sonarqube-persistentvolumeclaim.yaml -n ${env.K8S_NAMESPACE}
                        kubectl apply -f sonarqube-deployment.yaml -n ${env.K8S_NAMESPACE}
                        kubectl apply -f sonarqube-service.yaml -n ${env.K8S_NAMESPACE}

                        echo "SonarQube déployé. Attente du démarrage..."
                        sleep 60

                        # Vérifier l'état
                        kubectl get pods -l app=sonarqube -n ${env.K8S_NAMESPACE}
                        echo "URL SonarQube: http://localhost:30090"
                    """
                }
            }
        }

        stage('Deploy MySQL on K8S') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=/var/lib/jenkins/.kube/config

                        echo "=== Déploiement de MySQL sur K8S ==="

                        kubectl apply -f mysql-deployment.yaml -n ${env.K8S_NAMESPACE}

                        echo "MySQL déployé. Attente du démarrage..."
                        sleep 30

                        kubectl get pods -l app=mysql -n ${env.K8S_NAMESPACE}
                    """
                }
            }
        }

        stage('Verify SonarQube on K8S') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=/var/lib/jenkins/.kube/config

                        echo "=== Vérification de SonarQube sur K8S ==="

                        # Attendre que SonarQube soit prêt
                        echo "Attente de SonarQube..."
                        for i in {1..30}; do
                            if curl -s -f http://localhost:30090/api/system/status 2>/dev/null | grep -q "UP"; then
                                echo "✓ SonarQube est prêt!"
                                break
                            fi
                            echo "En attente... (\$i/30)"
                            sleep 10
                        done || echo "SonarQube prend du temps à démarrer"

                        # Vérifier l'accès
                        echo "Test d'accès à SonarQube..."
                        curl -s http://localhost:30090/api/system/status || echo "SonarQube non accessible"

                        # Vérifier notre projet
                        echo "Vérification du projet..."
                        curl -s "http://localhost:30090/api/projects/search?projects=student-management" | head -5 || echo "Impossible de vérifier le projet"
                    """
                }
            }
        }

        stage('Update and Deploy Spring Boot') {
            steps {
                script {
                    sh """
                        echo "=== Mise à jour et déploiement de Spring Boot ==="

                        # Mettre à jour l'image dans le fichier YAML
                        sed -i 's|image:.*mayamarzouki/student-management.*|image: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}|g' spring-deployment.yaml

                        # Déployer
                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        kubectl apply -f spring-deployment.yaml -n ${env.K8S_NAMESPACE}

                        echo "Spring Boot déployé. Attente du démarrage..."
                        sleep 30

                        kubectl get pods -l app=spring-boot-app -n ${env.K8S_NAMESPACE}
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=/var/lib/jenkins/.kube/config

                        echo "=== Vérification finale ==="

                        echo "1. État des pods:"
                        kubectl get pods -n ${env.K8S_NAMESPACE} -o wide

                        echo "2. Services:"
                        kubectl get svc -n ${env.K8S_NAMESPACE}

                        echo "3. Logs SonarQube:"
                        kubectl logs deployment/sonarqube-deployment -n ${env.K8S_NAMESPACE} --tail=5 2>/dev/null || echo "Pas de logs disponibles"

                        echo "4. URL SonarQube: http://localhost:30090"
                        echo "5. URL Application: http://localhost:30080/student"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build ${env.BUILD_NUMBER} réussi !"
            echo "🔗 SonarQube: http://localhost:30090"
            echo "🔗 Application Spring: http://localhost:30080/student"
        }
        failure {
            echo '❌ Build échoué!'
            sh '''
                echo "=== Débogage ==="
                export KUBECONFIG=/var/lib/jenkins/.kube/config

                echo "1. État des pods:"
                kubectl get pods -n devops

                echo "2. Détails SonarQube:"
                kubectl describe pod -l app=sonarqube -n devops 2>/dev/null | head -50 || true

                echo "3. Détails MySQL:"
                kubectl describe pod -l app=mysql -n devops 2>/dev/null | head -50 || true

                echo "4. Événements:"
                kubectl get events -n devops 2>/dev/null | tail -20 || true
            '''
        }
    }
}