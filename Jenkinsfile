pipeline {
    agent any


    stages {
        stage('Build') {
            steps {
                bat './gradlew sonarqube `-Dsonar.host.url=http://197.140.142.82:9000 `-Dsonar.login=80fd9868e62b454b44d672c3a4687f3bb0ba2a5e'
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