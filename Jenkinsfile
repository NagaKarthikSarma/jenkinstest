pipeline {
    agent any

    environment {
        // Your actual installations
        JAVA_HOME   = 'C:/Program Files/Amazon Corretto/jdk21.0.4_7'
        MAVEN_HOME  = 'C:/apache-maven-3.9.12'
        APP_PORT    = '8081'

        // Add Java + Maven to PATH
        PATH = "${MAVEN_HOME}/bin;${JAVA_HOME}/bin;${env.PATH}"
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
                    junit allowEmptyResults: true,
                          testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('Stop Existing Instance') {
            steps {
                bat """
                    FOR /F "tokens=5" %%P IN ('netstat -aon ^| findstr :${APP_PORT} ^| findstr LISTENING') DO (
                        taskkill /PID %%P /F
                    )
                """
            }
        }

        stage('Run Application') {
            steps {
                script {
                    def jarList = bat(
                        script: '@dir /b target\\*.jar | findstr /v sources | findstr /v javadoc',
                        returnStdout: true
                    ).trim().split("\\r?\\n").findAll { it?.trim() }

                    if (jarList.isEmpty()) {
                        error("No runnable JAR found!")
                    }

                    def jarFile = "target\\${jarList[0].trim()}"
                    echo "Running: ${jarFile}"

                    bat """
                        start /B java -jar "${jarFile}" --server.port=${APP_PORT} > app.log 2>&1
                        timeout /t 10
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                bat """
                    curl -f http://localhost:${APP_PORT}/actuator/health || (
                        curl -f http://localhost:${APP_PORT}/ || echo Health check failed
                    )
                """
            }
        }
    }

    post {
        success {
            echo "App running at: http://localhost:${APP_PORT}"
        }
        always {
            archiveArtifacts artifacts: 'target/*.jar, app.log', allowEmptyArchive: true
        }
    }
}
