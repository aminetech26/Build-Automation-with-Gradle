pipeline {
    agent any


    stages {
        stage('Build') {
            steps {
                bat './gradlew.bat clean build'
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