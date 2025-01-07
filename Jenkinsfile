pipeline {
    agent any

    environment {
        SMTP_USER = credentials('jenkins-smtp-user')
        SMTP_PASS = credentials('jenkins-smtp-pass')
    }

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
    }

    post {
            always {
                node('any') {
                    script {
                        currentBuild.result = currentBuild.result ?: 'SUCCESS'
                        if (currentBuild.result == 'SUCCESS') {
                            echo 'Sending success notifications...'
                            mail to: 'la_guerraiche@esi.dz',
                                 subject: "Build Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                                 body: """
                                    The build and deployment for ${env.JOB_NAME} #${env.BUILD_NUMBER} was successful.
                                    Build URL: ${env.BUILD_URL}
                                    Duration: ${currentBuild.durationString}
                                 """

                            slackSend(
                                color: '#00FF00',
                                channel: '#tp-gradle',
                                message: """
                                    :white_check_mark: Pipeline deployée avec succès!
                                    *Projet:* ${env.JOB_NAME}
                                    *Build:* ${env.BUILD_NUMBER}
                                    *Durée:* ${currentBuild.durationString}
                                """
                            )
                        } else {
                            echo 'Sending failure notifications...'
                            mail to: 'la_guerraiche@esi.dz',
                                 subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                                 body: """
                                    The build for ${env.JOB_NAME} #${env.BUILD_NUMBER} failed.
                                    Check the logs for details: ${env.BUILD_URL}console
                                    Duration: ${currentBuild.durationString}
                                 """

                            slackSend(
                                color: '#FF0000',
                                channel: '#tp-gradle',
                                message: """
                                    :x: Échec de la pipeline!
                                    *Projet:* ${env.JOB_NAME}
                                    *Build:* ${env.BUILD_NUMBER}
                                    *Durée:* ${currentBuild.durationString}
                                    *Voir les logs:* ${env.BUILD_URL}console
                                """
                            )
                        }
                    }

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
