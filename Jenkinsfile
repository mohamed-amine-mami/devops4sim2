pipeline {
    agent any
    
    tools {
        maven 'Maven-3'
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
                        kubectl create namespace ${env.K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
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

        /*  
        =======================================================================
        === 🚫 SonarQube COMPLETEMENT DÉSACTIVÉ (commenté proprement)       ===
        =======================================================================
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar -Dsonar.projectKey=student-management'
                }
            }
        }
        =======================================================================
        */

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

        /*  
        =======================================================================
        === 🚫 Tous les stages SonarQube K8S désactivés                      ===
        =======================================================================
        stage('Deploy SonarQube on K8S') { ... }
        stage('Verify SonarQube on K8S') { ... }
        =======================================================================
        */

        stage('Deploy MySQL on K8S') {
            steps {
                script {
                    sh """
                        export KUBECONFIG=/var/lib/jenkins/.kube/config

                        echo "=== Déploiement de MySQL sur K8S ==="
                        kubectl apply -f mysql-deployment.yaml -n ${env.K8S_NAMESPACE}
                        sleep 30
                        kubectl get pods -l app=mysql -n ${env.K8S_NAMESPACE}
                    """
                }
            }
        }

        stage('Update and Deploy Spring Boot') {
            steps {
                script {
                    sh """
                        echo "=== Mise à jour et déploiement de Spring Boot ==="

                        sed -i 's|image:.*mayamarzouki/student-management.*|image: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}|g' spring-deployment.yaml

                        export KUBECONFIG=/var/lib/jenkins/.kube/config
                        kubectl apply -f spring-deployment.yaml -n ${env.K8S_NAMESPACE}
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

                        kubectl get pods -n ${env.K8S_NAMESPACE} -o wide
                        kubectl get svc -n ${env.K8S_NAMESPACE}

                        echo "5. URL Application: http://localhost:30080/student"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build ${env.BUILD_NUMBER} réussi !"
            echo "🔗 Application Spring: http://localhost:30080/student"
        }
        failure {
            echo '❌ Build échoué!'
            sh '''
                echo "=== Débogage ==="
                export KUBECONFIG=/var/lib/jenkins/.kube/config
                kubectl get pods -n devops
                kubectl describe pod -l app=mysql -n devops 2>/dev/null | head -50 || true
                kubectl get events -n devops 2>/dev/null | tail -20 || true
            '''
        }
    }
}
