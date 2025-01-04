pipeline {
    agent any

    stages {
        stage('Test') {
            steps {
                // Run unit tests
                bat './gradlew.bat test'

                // Archive test results
                junit '**/build/test-results/test/*.xml'

                // Generate and archive Cucumber reports
                cucumber buildStatus: 'UNSTABLE',
                        fileIncludePattern: '**/cucumber.json',
                        jsonReportDirectory: 'build/reports/cucumber'
            }
        }

        stage('Code Analysis') {
            steps {
                // Run SonarQube analysis
                withSonarQubeEnv {
                    bat './gradlew.bat sonarqube'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    // Wait for quality gate result
                    timeout(time: 1, unit: 'HOURS') {
                        def qualityGate = waitForQualityGate()
                        if (qualityGate.status != 'OK') {
                            error "Pipeline aborted due to quality gate failure: ${qualityGate.status}"
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                // Generate JAR file
                bat './gradlew.bat clean build'

                // Generate documentation (assuming you're using Javadoc)
                bat './gradlew.bat javadoc'

                // Archive artifacts
                archiveArtifacts artifacts: [
                    '**/build/libs/*.jar',
                    '**/build/docs/**'
                ].join(','), fingerprint: true
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
            // Publish JaCoCo coverage report
            jacoco(
                execPattern: '**/build/jacoco/*.exec',
                classPattern: '**/build/classes/java/main',
                sourcePattern: '**/src/main/java'
            )

            // Clean workspace
            cleanWs()
        }
    }
}