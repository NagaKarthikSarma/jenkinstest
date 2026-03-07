pipeline {
    agent any 
    // tools {
    //     // These tools must be pre-configured in Jenkins Global Tool Configuration
    //     jdk 'jdk17' // Name of your JDK installation in Jenkins config
    //     maven 'maven3' // Name of your Maven installation in Jenkins config
    // }
    stages {
        stage('Pull Code from GitHub') {
            steps {
                // The checkout step automatically pulls the code from the SCM configured in the job
                checkout scm
            }
        }
        stage('Build') {
            steps {
                sh 'mvn -B -DskipTests clean package' // Builds the application and skips tests
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test' // Runs the unit tests
            }
            post {
                always {
                    // Archives the JUnit test reports
                    junit 'target/surefire-reports/*.xml' 
                }
            }
        }
        stage('Run Application') {
            steps {
                // This command runs the generated JAR file in the background (using '&')
                // Note: For production use, a more robust deployment strategy (e.g., Docker, Tomcat) is recommended
                sh 'nohup java -jar target/*.jar > app.log 2>&1 &' 
            }
        }
    }
    post {
        // Actions to run after the entire pipeline finishes
        always {
            // Optional: Archive the generated JAR artifact
            archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
            echo 'Pipeline finished.'
        }
    }
}
