pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        PROJECT_KEY  = 'psw-pipeline-base'
        SONAR_ORG    = 'esmeraldapv27'
        APP_HOST     = 'localhost'
        APP_PORT     = '8085'
    }

    stages {

        // Checkout
        stage('Checkout') {
            steps {
                checkout scm
                bat 'git log -1 --oneline'
                bat 'dir /b'
            }
        }

        // Build
        stage('Build') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    bat 'mvn -B clean package'
                }
            }
        }

        // Análisis con SonarCloud
        stage('Análisis con SonarCloud') {
            steps {
                catchError(buildResult: 'UNSTABLE', stageResult: 'UNSTABLE') {
                    script {
                        def sonarCredsExist = false
                        try {
                            withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                                sonarCredsExist = true
                                withSonarQubeEnv('SonarCloud') {
                                    bat 'mvn -B sonar:sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.organization=%SONAR_ORG% -Dsonar.projectKey=%PROJECT_KEY% -Dsonar.projectName="PSW Pipeline Base" -Dsonar.token=%SONAR_TOKEN% -Dsonar.java.binaries=target/classes'
                                }
                                timeout(time: 10, unit: 'MINUTES') {
                                    waitForQualityGate abortPipeline: false
                                }
                            }
                        } catch (e) {
                            echo "⚠️ SonarCloud omitido: ${e.message}"
                        }
                    }
                }
            }
        }

        // Pruebas con JMeter
        stage('Pruebas con JMeter') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    bat 'mvn -B verify -Djmeter.host=%APP_HOST% -Djmeter.puerto=%APP_PORT% -DskipTests=false'
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'target/jmeter/**', allowEmptyArchive: true, fingerprint: true
                }
            }
        }

    }

    post {
        success {
            echo "✅ Pipeline EXITOSO | Job: ${env.JOB_NAME} | Build: #${env.BUILD_NUMBER} | Rama: ${env.GIT_BRANCH ?: 'n/a'} | ${env.BUILD_URL}"
        }
        failure {
            echo "❌ Pipeline FALLIDO | Job: ${env.JOB_NAME} | Build: #${env.BUILD_NUMBER} | Rama: ${env.GIT_BRANCH ?: 'n/a'} | ${env.BUILD_URL}"
        }
        unstable {
            echo "⚠️ Pipeline INESTABLE | Job: ${env.JOB_NAME} | Build: #${env.BUILD_NUMBER} | Rama: ${env.GIT_BRANCH ?: 'n/a'} | ${env.BUILD_URL}"
        }
        always {
            junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
            archiveArtifacts artifacts: 'target/*.jar, target/site/jacoco/**', allowEmptyArchive: true, fingerprint: true
        }
        cleanup {
            cleanWs()
        }
    }

}