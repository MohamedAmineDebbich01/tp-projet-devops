pipeline {
    agent any

    tools {
        jdk 'JAVA_HOME'      // Nom du JDK dans Manage Jenkins > Tools
        maven 'M2_HOME'      // Nom de Maven dans Manage Jenkins > Tools
    }

    environment {
        SONARQUBE_SERVER = 'sonarqube-server'   // Nom du serveur SonarQube dans Configure System
    }

    stages {

        stage('Checkout') {
            steps {
                // Récupère le code selon la config du job (SCM Git, branche master)
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

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=tp-projet-AmineDebbich \
                          -Dsonar.projectName='TP Projet 2025 - Amine Debbich'
                    """
                }
            }
        }

        stage('Package JAR') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            }
        }
    }

    post {
        success {
            echo '✅ Build + Tests (si présents) + Sonar + JAR : OK'
        }
        failure {
            echo '❌ Échec du pipeline, vérifier la Console Output.'
        }
    }
}
