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
        bat '''
            setlocal enabledelayedexpansion

            rem === Find the first JAR in target ===
            set "JARFILE="
            for /f "delims=" %%F in ('dir /b /a:-d target\\*.jar') do (
                set "JARFILE=%%F"
                goto :found
            )

            echo No JAR found in target\\
            exit /b 1

            :found
            echo Found JAR: %JARFILE%

            rem Build the full path safely (handles spaces)
            set "JARPATH=%WORKSPACE%\\target\\%JARFILE%"

            rem Verify Java is on PATH (optional but helpful)
            where java || (echo "java" not found on PATH & exit /b 1)

            rem Launch detached with PowerShell using an ArgumentList array
            powershell -NoProfile -Command ^
              "$jar = [IO.Path]::GetFullPath('%JARPATH%'); " ^
              "Start-Process -FilePath 'java' -ArgumentList @('-jar', $jar) -WindowStyle Hidden"

            endlocal
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
