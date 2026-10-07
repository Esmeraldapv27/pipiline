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
        JTL          = 'target/jmeter/resultados.jtl'
        JM_REPORT    = 'target/jmeter/reporte'
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
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    withCredentials([string(credentialsId: 'sonarcloud-token', variable: 'SONAR_TOKEN')]) {
                        withSonarQubeEnv('SonarCloud') {
                            bat 'mvn -B sonar:sonar -Dsonar.host.url=https://sonarcloud.io -Dsonar.organization=%SONAR_ORG% -Dsonar.projectKey=%PROJECT_KEY% -Dsonar.projectName="PSW Pipeline Base" -Dsonar.token=%SONAR_TOKEN% -Dsonar.java.binaries=target/classes'
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
                    bat 'mkdir target\\jmeter 2>nul'
                    powershell script: '''
                        Start-Process -FilePath "java.exe" `
                            -ArgumentList "-jar", "target\\psw-pipeline-base-0.0.1-SNAPSHOT.jar" `
                            -RedirectStandardOutput "$PWD\\target\\app.log" `
                            -RedirectStandardError "$PWD\\target\\app.err.log" `
                            -PassThru | ForEach-Object { $_.Id | Set-Content -Path "target\\app.pid" }

                        $ok = $false
                        for ($i = 0; $i -lt 30; $i++) {
                            try {
                                $r = Invoke-WebRequest -Uri "http://localhost:8085/actuator/health" -UseBasicParsing -TimeoutSec 3
                                if ($r.StatusCode -eq 200) { $ok = $true; break }
                            } catch { }
                            Start-Sleep -Seconds 2
                        }
                        if (-not $ok) { Write-Error "[ERROR] La aplicacion no respondio en el health check"; exit 1 }
                    '''
                    bat 'jmeter -n -t jmeter\\pruebas.jmx -l %JTL% -e -o %JM_REPORT% -j target\\jmeter\\jmeter.log'
                    script {
                        def fallos = powershell(returnStdout: true, script: "if (Test-Path '${env.JTL}') { (Select-String -Path '${env.JTL}' -Pattern ',false,' | Measure-Object).Count } else { -1 }").trim()
                        echo "Muestras fallidas: ${fallos}"
                        if (fallos.toInteger() > 0) {
                            error("JMeter reportó ${fallos} muestras fallidas")
                        }
                    }
                }
            }
            post {
                always {
                    script {
                        if (fileExists('target/app.pid')) {
                            bat 'powershell -NoProfile -Command "if (Test-Path target\\app.pid) { $p = Get-Content target\\app.pid -Raw; Stop-Process -Id $p -Force -ErrorAction SilentlyContinue }"'
                        }
                    }
                    archiveArtifacts artifacts: 'target/jmeter/**, target/app.log', allowEmptyArchive: true, fingerprint: true
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