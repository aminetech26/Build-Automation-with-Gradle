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
            success {
                node('any') {
                    echo "Starting success notifications..."
                    emailext(
                        subject: "Pipeline Successful: ${currentBuild.fullDisplayName}",
                        body: """
                            La pipeline s'est terminée avec succès!
                            Détails:
                            - Projet: ${env.JOB_NAME}
                            - Build Numéro: ${env.BUILD_NUMBER}
                            - Status: SUCCESS
                            - Durée: ${currentBuild.durationString}
                        """,
                        to: "amine.fewd@gmail.com",
                        from: "la_guerraiche@esi.dz",
                        mimeType: 'text/html'
                    )
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
                }
            }

            failure {
                node('any') {
                    echo "Starting failure notifications..."
                    emailext(
                        subject: "Pipeline Failed: ${currentBuild.fullDisplayName}",
                        body: """
                            La pipeline a échoué!
                            Détails:
                            - Projet: ${env.JOB_NAME}
                            - Build Numéro: ${env.BUILD_NUMBER}
                            - Status: FAILURE
                            - Durée: ${currentBuild.durationString}
                        """,
                        to: "amine.fewd@gmail.com",
                        from: "la_guerraiche@esi.dz",
                        mimeType: 'text/html',
                        attachLog: true,
                        compressLog: true
                    )
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

            always {
                node('any') {
                    echo "Executing always block..."
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
