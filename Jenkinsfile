pipeline {

    agent any

    tools {
        maven 'Maven3'
        jdk 'JDK17'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        PROJECT_KEY  = 'psw-pipeline-base'
        SONAR_ORG    = 'esmeraldapv27'
        APP_HOST    = 'localhost'
        APP_PORT    = '8085'
        JTL         = 'target/jmeter/resultados.jtl'
        JM_REPORT   = 'target/jmeter/reporte'
        SLACK_CHANNEL = '#ci-psw'
    }

    stages {

        // Checkout
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
                sh 'ls -la'
            }
        }

        // Build
        stage('Build') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh 'mvn -B clean package'
                }
            }
        }

        // Análisis con SonarCloud
        stage('Análisis con SonarCloud') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                        withSonarQubeEnv('SonarCloud') {
                            sh 'mvn -B sonar:sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.organization=${SONAR_ORG} -Dsonar.projectKey=${PROJECT_KEY} -Dsonar.projectName="PSW Pipeline Base" -Dsonar.token=${SONAR_TOKEN} -Dsonar.java.binaries=target/classes'
                        }
                    }
                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: false
                    }
                }
            }
        }

        // Pruebas con JMeter
        stage('Pruebas con JMeter') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh 'mkdir -p target/jmeter'
                    sh '''
                        nohup java -jar target/psw-pipeline-base-*.jar > target/app.log 2>&1 &
                        echo $! > target/app.pid
                        echo "Iniciando la aplicacion..."
                        for i in $(seq 1 30); do
                            if curl -sf http://localhost:8085/actuator/health > /dev/null; then
                                echo "Aplicacion UP"; break
                            fi
                            sleep 2
                        done
                        curl -sf http://localhost:8085/actuator/health > /dev/null || { echo "La aplicacion no respondio"; exit 1; }
                    '''
                    sh 'jmeter -n -t jmeter/pruebas.jmx -l ${JTL} -e -o ${JM_REPORT} -j target/jmeter/jmeter.log'
                    script {
                        def cmd = "awk -F',' 'NR==1 {for(i=1;i<=NF;i++) if(\$i==\"success\") c=i; next} \$c==\"false\" {f++} END {print f+0}' ${env.JTL}"
                        def fallos = sh(script: cmd, returnStdout: true).trim()
                        echo "Muestras fallidas: ${fallos}"
                        if (fallos.toInteger() > 0) {
                            error("JMeter reportó ${fallos} muestras fallidas")
                        }
                    }
                }
            }
            post {
                always {
                    sh 'if [ -f target/app.pid ]; then kill $(cat target/app.pid) 2>/dev/null || true; fi'
                    archiveArtifacts artifacts: 'target/jmeter/**, target/app.log', allowEmptyArchive: true, fingerprint: true
                }
            }
        }

    }

    post {
        success {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: "*Pipeline:* ${env.JOB_NAME}\n*Build:* #${env.BUILD_NUMBER}\n*Estado:* EXITOSO\n*Rama:* ${env.GIT_BRANCH ?: 'n/a'}\n*Detalles:* ${env.BUILD_URL}"
            )
        }
        failure {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: "*Pipeline:* ${env.JOB_NAME}\n*Build:* #${env.BUILD_NUMBER}\n*Estado:* FALLIDO (${currentBuild.currentResult})\n*Rama:* ${env.GIT_BRANCH ?: 'n/a'}\n*Detalles:* ${env.BUILD_URL}"
            )
        }
        unstable {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'warning',
                message: "*Pipeline:* ${env.JOB_NAME}\n*Build:* #${env.BUILD_NUMBER}\n*Estado:* INESTABLE\n*Rama:* ${env.GIT_BRANCH ?: 'n/a'}\n*Detalles:* ${env.BUILD_URL}"
            )
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
