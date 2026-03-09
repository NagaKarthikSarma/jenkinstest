pipeline {
    agent any

    environment {
        // Your actual installations
        JAVA_HOME  = 'C:/Program Files/Amazon Corretto/jdk21.0.4_7'
        MAVEN_HOME = 'C:/apache-maven-3.9.12'
        APP_PORT   = '8081'
        PATH = "${MAVEN_HOME}/bin;${JAVA_HOME}/bin;${env.PATH}"
        HEALTH_URL = "http://localhost:${APP_PORT}/actuator/health"
        FALLBACK_URL = "http://localhost:${APP_PORT}/"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/NagaKarthikSarma/jenkinstest.git'
            }
        }

        stage('Verify Tools') {
            steps {
                bat 'java -version'
                bat 'mvn -version'
            }
        }

        stage('Build') {
            steps {
                script {
                    if (fileExists('mvnw.cmd')) {
                        bat 'mvnw.cmd clean package -DskipTests'
                    } else {
                        bat 'mvn clean package -DskipTests'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    if (fileExists('mvnw.cmd')) {
                        bat 'mvnw.cmd test'
                    } else {
                        bat 'mvn test'
                    }
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Stop Existing Instance') {
            steps {
                echo "Stopping any existing instance on port ${env.APP_PORT}..."
                bat """
                    @echo off
                    setlocal
                    set "PORT=${APP_PORT}"
                    rem Check if anything is listening; if not, skip
                    netstat -aon ^| findstr :%PORT% ^| findstr LISTENING >nul 2>&1
                    if errorlevel 1 (
                        echo No process is listening on port %PORT%.
                    ) else (
                        for /f "tokens=5" %%P in ('netstat -aon ^| findstr :%PORT% ^| findstr LISTENING') do (
                            echo Killing PID %%P on port %PORT%
                            taskkill /PID %%P /F >nul 2>&1
                        )
                    )
                    exit /b 0
                """
            }
        }

 stage('Run Application') {
    steps {
        script {
            // Find runnable jar (exclude sources/javadoc)
            def jarList = bat(
                script: '@dir /b target\\*.jar | findstr /v sources | findstr /v javadoc',
                returnStdout: true
            ).trim().split("\\r?\\n").findAll { it?.trim() }

            if (!jarList || jarList.isEmpty()) {
                error('No runnable JAR found under target\\')
            }

            def jarFile = "target\\${jarList[0].trim()}"
            echo "Running: ${jarFile}"

            // Start app and wait for it to exit. When Actuator shuts it down, this stage will end.
            bat """
                @echo off
                if exist app.pid del /f /q app.pid

                rem Start the Java app, capture PID, redirect logs, and wait for exit
                powershell -NoProfile -Command ^
                  "$args = '-jar', '${jarFile}', '--server.port=%APP_PORT%'; ^
                   $p = Start-Process 'java' -ArgumentList $args -PassThru -WindowStyle Hidden ^
                        -RedirectStandardOutput 'app.log' -RedirectStandardError 'app.log'; ^
                   $p.Id | Out-File -FilePath 'app.pid' -Encoding ascii; ^
                   Write-Host ('Started PID ' + $p.Id + ' on port %APP_PORT%'); ^
                   try { Wait-Process -Id $p.Id } finally { Write-Host 'Application process exited.' }"
            """
        }
    }
}

        stage('Health Check') {
            steps {
                script {
                    // Poll health for up to 60s (12 * 5s)
                    def ok = false
                    for (int i = 1; i <= 12; i++) {
                        def rc = bat(script: "curl -sf ${HEALTH_URL}", returnStatus: true)
                        if (rc == 0) { ok = true; break }
                        // Try root if actuator not present
                        rc = bat(script: "curl -sf ${FALLBACK_URL}", returnStatus: true)
                        if (rc == 0) { ok = true; break }
                        echo "Health not ready yet... retry ${i}/12"
                        // 5s sleep (10 pings ~9s; 6 pings ~5s)
                        bat 'ping -n 6 127.0.0.1 >nul'
                    }
                    if (!ok) {
                        echo 'Health check failed. Showing last 100 lines of app.log:'
                        bat 'powershell -NoProfile -Command "Get-Content -Path app.log -Tail 100 | Out-String"'
                        error('Application did not become healthy in time.')
                    } else {
                        echo "✅ Application is healthy on port ${APP_PORT}"
                    }
                }
            }
        }
    }

    post {
        success {
            echo "App is running at: http://localhost:${APP_PORT}"
        }
        always {
            archiveArtifacts artifacts: 'target/*.jar, app.log', allowEmptyArchive: true
        }
    }
}
