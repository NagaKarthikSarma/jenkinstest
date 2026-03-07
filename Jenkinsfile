pipeline {
    // Use a Windows-based agent. Ensure your Jenkins agent is configured as a Windows node.
    agent any

    stages {
        stage('Pull Code from GitHub') {
            steps {
                // The git step automatically handles cloning the repository based on the job configuration.
                git branch: 'main', url: 'https://github.com/NagaKarthikSarma/jenkinstest.git'
            }
        }

        stage('Build Spring Boot App') {
            steps {
                // Use bat or powershell for Windows shell commands
                // Run Maven clean install to build the project and generate the JAR file
                bat 'mvn clean install -DskipTests'
            }
        }

        stage('Run Application') {
            steps {
                // Stop any previously running instance of the app (optional but recommended for redeployment)
                // This is a basic example; proper service management might be needed for production
                bat 'taskkill /F /IM java.exe || ECHO No running processes found'

                // Run the Spring Boot JAR file
                // The JAR file is typically located in the 'target' directory after the build stage
                bat 'java -jar target/*.jar'
            }
        }
    }
}
