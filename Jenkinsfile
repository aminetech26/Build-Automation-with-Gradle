pipeline {
    agent any

    stages {
        stage('Test') {
            steps {
                bat './gradlew.bat compileJava'
                bat './gradlew.bat clean test'
                junit '**/build/test-results/test/*.xml'
                cucumber(
                    fileIncludePattern: '**/cucumber.json',
                    jsonReportDirectory: 'build/reports/cucumber'
                )
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
                    withCredentials([usernamePassword(credentialsId: '5ff6be52-9749-404b-b6ea-a1f8d9654e3c', usernameVariable: 'username', passwordVariable: 'password')]) {
                        bat './gradlew.bat publish'
                    }
                }
            }
        }

        stage('Notifications') {
            steps {
                script {
                    currentBuild.result = currentBuild.result ?: 'SUCCESS'

                    if (currentBuild.result == 'SUCCESS') {
                        echo 'Sending success notifications...'
                        mail to: 'la_guerraiche@esi.dz',
                             subject: "Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                             body: "The build and deployment for ${env.JOB_NAME} #${env.BUILD_NUMBER} was successful."

                        slackSend channel: '#tp-gradle',
                                  color: 'good',
                                  message: """
                                      :white_check_mark: *Build Success!*
                                      *Project:* ${env.JOB_NAME}
                                      *Build Number:* ${env.BUILD_NUMBER}
                                      *Duration:* ${currentBuild.durationString}
                                      *View Logs:* <${env.BUILD_URL}|Console Output>
                                  """
                    } else {
                        echo 'Sending failure notifications...'
                        mail to: 'la_guerraiche@esi.dz',
                             subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                             body: "The build for ${env.JOB_NAME} #${env.BUILD_NUMBER} failed. Check the logs for details."

                        slackSend channel: '#tp-gradle',
                                  color: 'danger',
                                  message: """
                                      :x: *Build Failed!*
                                      *Project:* ${env.JOB_NAME}
                                      *Build Number:* ${env.BUILD_NUMBER}
                                      *Duration:* ${currentBuild.durationString}
                                      *View Logs:* <${env.BUILD_URL}|Console Output>
                                  """
                    }
                }
            }
        }

    }

    post {
            always {
                node('any') {
                    jacoco(
                        execPattern: '**/build/jacoco/*.exec',
                        classPattern: '**/build/classes/java/main',
                        sourcePattern: '**/src/main/java'
                    )
                    cleanWs()
                }
            }
        }
}
