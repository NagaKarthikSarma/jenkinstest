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
            // 1. Locate the JAR (Excluding sources/javadoc)
            def jarList = bat(
                script: '@dir /b target\\*.jar | findstr /v sources | findstr /v javadoc',
                returnStdout: true
            ).trim().split("\\r?\\n").findAll { it?.trim() }

            if (!jarList) {
                error('No runnable JAR found under target\\')
            }

            def jarFile = "target\\${jarList[0].trim()}"
            def processTitle = "Jenkins_App_${env.BUILD_ID}"
            
            echo "Starting ${jarFile} on port ${env.APP_PORT}..."

            // 2. Launch and Monitor
            bat """
                @echo off
                :: Start the app with a specific title in the background
                :: Redirecting output to app.log for debugging
                start "${processTitle}" /B java -jar "${jarFile}" --server.port=${env.APP_PORT} > app.log 2>&1

                echo Application is booting. Monitoring process: ${processTitle}
                echo To stop this stage, trigger: http://localhost:${env.APP_PORT}/actuator/shutdown

                :monitor
                :: Check if the process with our unique title is still in the task list
                tasklist /V /FI "WINDOWTITLE eq ${processTitle}" | findstr /i "java.exe" >nul
                
                if %ERRORLEVEL% equ 0 (
                    :: Process still exists. Wait 10 seconds (ping -n 11) and check again.
                    ping -n 11 127.0.0.1 >nul
                    goto monitor
                )

                echo [INFO] Application process has terminated (Actuator shutdown detected).
                echo [INFO] Final logs from app.log:
                type app.log
            """
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
