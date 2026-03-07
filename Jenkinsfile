pipeline {
    agent any
    // If you configured tools in Jenkins, you can uncomment and match their names:
    // tools {
    //   jdk 'jdk21'
    //   maven 'maven3'
    // }

    stages {
        stage('Pull Code from GitHub') {
            steps {
                // Pulls the code from the job's configured SCM
                checkout scm
            }
        }

        stage('Build') {
            steps {
                script {
                    // Use Maven Wrapper if present; fall back to mvn
                    def skip = '-DskipTests'
                    if (fileExists('mvnw.cmd')) {
                        bat "mvnw -B -U clean package ${skip}"
                    } else {
                        bat "mvn -B -U clean package ${skip}"
                    }
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    if (fileExists('mvnw.cmd')) {
                        bat "mvnw -B test"
                    } else {
                        bat "mvn -B test"
                    }
                }
            }
            post {
                always {
                    // Publish JUnit test results if present
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Run Application') {
            steps {
                script {
                    // Find the packaged JAR (exclude sources/javadoc)
                    def jarPath = bat(
                        returnStdout: true,
                        script: """
                          powershell -NoProfile -Command ^
                            "(Get-ChildItem -Path 'target\\*.jar' -ErrorAction SilentlyContinue | `
                              Where-Object { \$_ .Name -notmatch 'sources|javadoc' } | `
                              Sort-Object LastWriteTime -Descending | `
                              Select-Object -First 1).FullName"
                        """
                    ).trim()

                    if (!jarPath) {
                        error "No runnable JAR found in target/. Ensure Spring Boot builds a fat JAR."
                    }

                    echo "Starting: ${jarPath}"

                    // Start Spring Boot app in background, redirect output to app.log, and capture PID
                    bat """
                      powershell -NoProfile -Command ^
                        "\$log = Join-Path (Get-Location) 'app.log'; ^
                         if (Test-Path \$log) { Remove-Item \$log -Force }; ^
                         \$args = @('-jar','${jarPath.replace('\\','/')}'); ^
                         \$p = Start-Process -FilePath 'java' -ArgumentList \$args -PassThru -WindowStyle Hidden -RedirectStandardOutput \$log -RedirectStandardError \$log; ^
                         Set-Content -Path app.pid -Value \$p.Id; ^
                         'Started PID: ' + \$p.Id | Out-File -FilePath \$log -Append"
                    """

                    // Optional: wait briefly and show last lines of the log
                    bat "powershell -NoProfile -Command \"Start-Sleep -Seconds 5; Get-Content .\\app.log -Tail 60 -ErrorAction SilentlyContinue\""
                }
            }
        }
    }

    post {
        always {
            // Archive generated JAR(s)
            archiveArtifacts artifacts: 'target\\*.jar', fingerprint: true
            echo 'Pipeline finished.'
        }
    }
}
