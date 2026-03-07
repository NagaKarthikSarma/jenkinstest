pipeline {
    agent any // Specifies that the pipeline can run on any available agent, including a Windows machine
    // tools {
    //     // Ensure you configure these tool names in Manage Jenkins > Global Tool Configuration
    //     maven 'Maven 3.x' 
    //     jdk 'JDK 17' 
    // }
    stages {
        stage('Checkout Code') {
            steps {
                // Fetches the code from the specified GitHub repository
                git url: 'https://github.com', 
                    branch: 'main' // Replace 'main' with your target branch
            }
        }

        stage('Build Project') {
            steps {
                // Runs Maven clean package command to build the JAR file
                bat 'mvn clean package -DskipTests' //
            }
        }

        stage('Run Application in New Window') {
            steps {
                // Finds the generated JAR file path dynamically
                script {
                    def jarFile = findFiles(glob: 'target/*.jar')[0].path
                    // Launches the application in a new, detached CMD window on Windows
                    // The 'start cmd /k' command opens a new window and keeps it open (/k) after running the command
                    bat "start \"SpringBoot App\" cmd /k java -jar %WORKSPACE%\\\\${jarFile}"
                }
            }
        }
    }
}
