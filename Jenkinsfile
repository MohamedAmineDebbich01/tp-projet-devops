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
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Clean') {
            steps {
                sh 'mvn clean'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        // ✅ Sonar NON BLOQUANT (si Sonar down -> stage UNSTABLE, pipeline continue)
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

        // ✅ Push Docker NON BLOQUANT (si DockerHub 502 -> UNSTABLE, pipeline OK)
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
    }

    post {
        success {
            echo '✅ Pipeline OK (Clean + Compile + Sonar non bloquant + Package + Docker Build + Docker Push non bloquant)'
        }
        unstable {
            echo '⚠️ Pipeline UNSTABLE : Sonar ou Docker Push a échoué mais le pipeline continue.'
        }
        failure {
            echo '❌ Pipeline FAILED : étape bloquante a échoué.'
        }
        always {
            sh '''
              echo "---- Docker images (top 20) ----"
              docker images | head -n 20 || true
            '''
        }
    }
}
