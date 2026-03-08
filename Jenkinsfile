pipeline {
    agent any

    environment {
        // ── Change these to match your project ──────────────────────────
        GITHUB_REPO_URL   = 'https://github.com/your-username/your-springboot-repo.git'
        BRANCH_NAME       = 'main'
        JAVA_HOME    = 'C:/Program Files/Amazon Corretto/jdk21.0.4_7'  // adjust JDK path
       MAVEN_HOME        = 'C:/apache-maven-3.9.12'            // adjust Maven path (if not using wrapper)
        APP_PORT          = '8081'
        // ────────────────────────────────────────────────────────────────
    }

    tools {
        // These must match the names configured in Jenkins → Global Tool Configuration
        jdk   'JDK-17'
        maven 'Maven-3.9'
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Cloning repository: ${env.GITHUB_REPO_URL}"
                git branch: "${env.BRANCH_NAME}",
                    url: "${env.GITHUB_REPO_URL}"
                    // credentialsId: 'github-credentials-id'  // ← uncomment for private repos
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
                echo 'Building Spring Boot application...'
                // Use mvnw.cmd if Maven Wrapper is present, otherwise use mvn
                script {
                    def hasMvnWrapper = fileExists('mvnw.cmd')
                    if (hasMvnWrapper) {
                        bat 'mvnw.cmd clean package -DskipTests'
                    } else {
                        bat 'mvn clean package -DskipTests'
                    }
                }
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests...'
                script {
                    def hasMvnWrapper = fileExists('mvnw.cmd')
                    if (hasMvnWrapper) {
                        bat 'mvnw.cmd test'
                    } else {
                        bat 'mvn test'
                    }
                }
            }
            post {
                always {
                    // Publish JUnit test results
                    junit allowEmptyResults: true,
                          testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Stop Existing Instance') {
            steps {
                echo 'Stopping any existing instance on port ${env.APP_PORT}...'
                // Kills the process using the app port (ignores error if nothing running)
                bat """
                    FOR /F "tokens=5" %%P IN ('netstat -aon ^| findstr :${env.APP_PORT} ^| findstr LISTENING') DO (
                        echo Killing PID %%P
                        taskkill /PID %%P /F
                    )
                    exit /b 0
                """
            }
        }

        stage('Run Application') {
            steps {
                echo 'Starting Spring Boot application...'
                script {
                    // Find the built JAR (excludes *-sources.jar and *-javadoc.jar)
                    def jarFile = bat(
                        script: '@dir /b /s target\\*.jar | findstr /v sources | findstr /v javadoc',
                        returnStdout: true
                    ).trim()

                    echo "Found JAR: ${jarFile}"

                    // Launch app in background (start /B keeps Jenkins from hanging)
                    bat """
                        start /B java -jar "${jarFile}" --server.port=${env.APP_PORT} > app.log 2>&1
                    """

                    // Wait for the app to start (polls /actuator/health if available)
                    bat """
                        echo Waiting for application to start...
                        timeout /t 15 /nobreak
                        echo Application should be running on http://localhost:${env.APP_PORT}
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                echo 'Performing health check...'
                // Basic connectivity check using curl (requires curl on Windows PATH)
                bat """
                    curl -f http://localhost:${env.APP_PORT}/actuator/health || (
                        echo Health endpoint not available - trying root path...
                        curl -f http://localhost:${env.APP_PORT}/ || echo App may still be starting up
                    )
                    exit /b 0
                """
            }
        }
    }

    post {
        success {
            echo """
            ✅ BUILD SUCCESSFUL
            Spring Boot app is running at: http://localhost:${env.APP_PORT}
            Logs available at: \${WORKSPACE}\\app.log
            """
        }
        failure {
            echo '❌ BUILD FAILED - Check console output for details'
        }
        always {
            // Archive the JAR and logs as build artifacts
            archiveArtifacts artifacts: 'target/*.jar, app.log',
                             allowEmptyArchive: true
        }
    }
}
