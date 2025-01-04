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
                script {
                    // Vérifier si des tests Cucumber ont échoué
                    def cucumberReports = findFiles(glob: '**/build/reports/cucumber/cucumber.json')
                    if (cucumberReports.length == 0) {
                        error "Aucun rapport Cucumber trouvé. Les tests ont peut-être échoué."
                    }
                }
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
            script {
                echo "Starting success notifications..."

                // Email notification for success
                try {
                    echo "Attempting to send success email..."
                    emailext (
                        subject: "Pipeline Successful: ${currentBuild.fullDisplayName}",
                        body: """
                            La pipeline s'est terminée avec succès!

                            Détails:
                            - Projet: ${env.JOB_NAME}
                            - Build Numéro: ${env.BUILD_NUMBER}
                            - Status: SUCCESS
                            - Durée: ${currentBuild.durationString}

                            Debug Info:
                            - SMTP User: ${SMTP_USER}
                            - From: la_guerraiche@esi.dz
                            - To: amine.fewd@gmail.com
                            - Time: ${new Date().format("yyyy-MM-dd HH:mm:ss")}

                            Voir les détails complets: ${env.BUILD_URL}
                        """,
                        to: "amine.fewd@gmail.com",
                        from: "la_guerraiche@esi.dz",
                        mimeType: 'text/html',
                        attachLog: false,
                        compressLog: false
                    )
                    echo "Success email sent successfully"
                } catch (e) {
                    echo "Failed to send success email: ${e.getMessage()}"
                    e.printStackTrace()
                }

                // Slack notification for success
                try {
                    echo "Attempting to send Slack success message..."
                    slackSend (
                        color: '#00FF00',
                        channel: '#tp-gradle',
                        message: """
                            :white_check_mark: Pipeline deployée avec succès!
                            *Projet:* ${env.JOB_NAME}
                            *Build:* ${env.BUILD_NUMBER}
                            *Durée:* ${currentBuild.durationString}
                        """
                    )
                    echo "Slack success message sent successfully"
                } catch (e) {
                    echo "Failed to send Slack success message: ${e.getMessage()}"
                    e.printStackTrace()
                }
            }
        }

        failure {
            script {
                echo "Starting failure notifications..."

                // Email notification for failure
                try {
                    echo "Attempting to send failure email..."
                    emailext (
                        subject: "Pipeline Failed: ${currentBuild.fullDisplayName}",
                        body: """
                            La pipeline a échoué!

                            Détails:
                            - Projet: ${env.JOB_NAME}
                            - Build Numéro: ${env.BUILD_NUMBER}
                            - Status: FAILURE
                            - Durée: ${currentBuild.durationString}

                            Debug Info:
                            - SMTP User: ${SMTP_USER}
                            - From: la_guerraiche@esi.dz
                            - To: amine.fewd@gmail.com
                            - Time: ${new Date().format("yyyy-MM-dd HH:mm:ss")}

                            Voir les logs: ${env.BUILD_URL}console
                        """,
                        to: "amine.fewd@gmail.com",
                        from: "la_guerraiche@esi.dz",
                        mimeType: 'text/html',
                        attachLog: true,
                        compressLog: true
                    )
                    echo "Failure email sent successfully"
                } catch (e) {
                    echo "Failed to send failure email: ${e.getMessage()}"
                    e.printStackTrace()
                }

                // Slack notification for failure
                try {
                    echo "Attempting to send Slack failure message..."
                    slackSend (
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
                    echo "Slack failure message sent successfully"
                } catch (e) {
                    echo "Failed to send Slack failure message: ${e.getMessage()}"
                    e.printStackTrace()
                }
            }
        }

        always {
            echo "Executing always block..."
            try {
                jacoco(
                    execPattern: '**/build/jacoco/*.exec',
                    classPattern: '**/build/classes/java/main',
                    sourcePattern: '**/src/main/java'
                )
                echo "JaCoCo report generated successfully"
            } catch (e) {
                echo "Failed to generate JaCoCo report: ${e.getMessage()}"
            }

            try {
                cleanWs()
                echo "Workspace cleaned successfully"
            } catch (e) {
                echo "Failed to clean workspace: ${e.getMessage()}"
            }
        }
    }
}