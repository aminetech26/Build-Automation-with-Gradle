pipeline {
    agent any

    stages {
        stage('Compile') {
            steps {
                bat './gradlew.bat compileJava'
            }
        }

        stage('Test') {
            steps {
                bat './gradlew.bat clean test'
                junit '**/build/test-results/test/*.xml'
                cucumber buildStatus: 'UNSTABLE',
                        fileIncludePattern: '**/cucumber.json',
                        jsonReportDirectory: 'build/reports/cucumber'
            }
        }

        stage('Code Analysis') {
            environment {
                SONAR_HOST_URL = 'http://197.140.142.82:9000'
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    bat """
                        ./gradlew.bat sonarqube \
                        -Dsonar.host.url=${SONAR_HOST_URL} \
                        -Dsonar.gradle.skipCompile=true
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
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
                bat './gradlew.bat clean build'
                bat './gradlew.bat javadoc'
                archiveArtifacts artifacts: [
                    '**/build/libs/*.jar',
                    '**/build/docs/**'
                ].join(','), fingerprint: true
            }
        }

        stage('Deploy') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: '957281a2-489e-4817-9caa-105ddeb04dc6', usernameVariable: 'username', passwordVariable: 'password')]) {
                        bat './gradlew.bat publish'
                    }
                }
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
            jacoco(
                execPattern: '**/build/jacoco/*.exec',
                classPattern: '**/build/classes/java/main',
                sourcePattern: '**/src/main/java'
            )
            cleanWs()
        }
    }
}