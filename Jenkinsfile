pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonartoken')
    }

    stages {
        stage('Build') {
            steps {
                bat './gradlew.bat clean build'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                bat './gradlew sonarqube -Dsonar.login=${SONAR_TOKEN}'
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                script {
                    def qualityGate = waitForQualityGate()

                    if (qualityGate.status != 'OK') {
                        error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
                    }
                }
            }
        }

        stage('Artifact') {
            steps {
                archiveArtifacts artifacts: '**/build/libs/*.jar', fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'Build completed successfully!'
        }

        failure {
            echo 'Build failed. Please check the logs.'
        }

        always {
            cleanWs()
        }
    }
}