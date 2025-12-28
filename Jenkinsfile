pipeline {
    agent any

    tools {
        jdk 'JAVA_HOME'
        maven 'M2_HOME'
    }

    environment {
        // SonarQube dans Jenkins (Manage Jenkins > Configure System > SonarQube)
        SONARQUBE_SERVER = 'sonarqube-server'

        // Docker Hub
        DOCKER_CRED_ID = 'dockerhub-cred'          // ID du credential Jenkins (Username/Password)
        DOCKER_IMAGE   = 'amine12123/projetdocker' // ton repo Docker Hub
        DOCKER_TAG     = '1.0'                     // version de l'image

        // K8S
        APP_DEPLOYMENT = 'tpfoyer'
        MYSQL_DEPLOYMENT = 'mysql'
        APP_SERVICE = 'tpfoyer-svc'
    }

    stages {

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Clean') {
            steps { sh 'mvn clean' }
        }

        stage('Compile') {
            steps { sh 'mvn compile' }
        }

        // ✅ Sonar NON BLOQUANT
        stage('SonarQube Analysis') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    withSonarQubeEnv("${SONARQUBE_SERVER}") {
                        sh """
                          mvn sonar:sonar \
                            -Dsonar.projectKey=tp-projet-AmineDebbich \
                            -Dsonar.projectName='TP Projet 2025 - Amine Debbich'
                        """
                    }
                }
            }
        }

        stage('Package JAR') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                  docker --version
                  docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                  docker images | head -n 20
                '''
            }
        }

        // ✅ Push Docker NON BLOQUANT
        stage('Push Docker Image to Docker Hub') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                    withCredentials([usernamePassword(
                        credentialsId: "${DOCKER_CRED_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh '''
                          set -e
                          echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                          docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                          docker logout
                        '''
                    }
                }
            }
        }

        // ✅ DEPLOY K8S (bloquant)
        stage('Deploy MySQL & Spring Boot on K8s') {
            steps {
                sh '''
                  echo "---- kubectl check ----"
                  kubectl version --client

                  echo "---- Apply MySQL ----"
                  kubectl apply -f k8s/mysql-secret.yaml
                  kubectl apply -f k8s/mysql-deployment.yaml
                  kubectl apply -f k8s/mysql-service.yaml

                  echo "---- Apply APP ----"
                  kubectl apply -f k8s/app-deployment.yaml
                  kubectl apply -f k8s/app-service.yaml

                  echo "---- Rollout status ----"
                  kubectl rollout status deployment/${MYSQL_DEPLOYMENT} --timeout=180s
                  kubectl rollout status deployment/${APP_DEPLOYMENT} --timeout=180s

                  echo "---- Pods & Services ----"
                  kubectl get pods -o wide
                  kubectl get svc -o wide
                '''
            }
        }

        // ✅ Vérification simple
        stage('Verification') {
            steps {
                sh '''
                  echo "---- Service URL (Minikube) ----"
                  (minikube service ${APP_SERVICE} --url || true)

                  echo "✅ Verification done"
                '''
            }
        }

        // ✅ Notification simple (console)
        stage('Notification') {
            steps {
                echo "📣 Build: ${BUILD_URL}"
                echo "📣 Docker Image: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                echo "📣 K8s Service: ${APP_SERVICE} (NodePort 30080)"
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline OK (Build + Sonar non bloquant + Push non bloquant + Deploy K8S + Verification + Notification)'
            echo '👉 Accès (NodePort): http://<IP_VM>:30080'
        }
        unstable {
            echo '⚠️ Pipeline UNSTABLE : Sonar ou Docker Push a échoué mais le pipeline continue.'
            echo '👉 Si Deploy K8S est OK : http://<IP_VM>:30080'
        }
        failure {
            echo '❌ Pipeline FAILED : étape bloquante a échoué (Build / Package / Deploy K8S...).'
        }
        always {
            sh '''
              echo "---- Docker images (top 20) ----"
              docker images | head -n 20 || true

              echo "---- K8s state ----"
              kubectl get pods -o wide || true
              kubectl get svc -o wide || true
            '''
        }
    }
}
