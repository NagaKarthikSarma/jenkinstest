pipeline {
    agent any // Ensure you have a Windows agent with this label

    // Uncomment if you configured Maven/JDK in Global Tool Configuration:
    // tools {
    //     maven 'Maven 3.x'
    //     jdk   'JDK 17'
    // }

    stages {
        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/NagaKarthikSarma/jenkinstest.git', branch: 'main'
            }
        }

        stage('Build Project') {
            steps {
                // If Maven is not on PATH, configure "tools" above or call it via full path.
                bat 'mvn -v'
                bat 'mvn clean package -DskipTests'
            }
        }

        stage('Run Application in New Window (Detached)') {
            steps {
                // Batch finds the first JAR under target and stores in JARFILE env var
                bat '''
                    setlocal enabledelayedexpansion
                    for /f "delims=" %%F in ('dir /b /a:-d target\\*.jar') do (
                        set JARFILE=%%F
                        goto :found
                    )
                    echo No JAR found in target\\
                    exit /b 1
                    :found
                    echo Found JAR: %JARFILE%

                    rem Use PowerShell to start the process detached (no console kept open)
                    powershell -NoProfile -Command ^
                      "Start-Process -FilePath 'java' -ArgumentList '-jar \"'%WORKSPACE%\\target\\%JARFILE%'\"' -WindowStyle Hidden"
                '''
            }
        }
    }

    post {
        failure {
            echo 'Build failed. Check the console log for details.'
        }
    }
}
