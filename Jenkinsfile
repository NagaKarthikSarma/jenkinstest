pipeline {
  agent any

  parameters {
    string(name: 'REPO_URL', defaultValue: 'https://github.com/NagaKarthikSarma/jenkinstest.git', description: 'Repository URL (if using inline pipeline)')
    string(name: 'BRANCH',    defaultValue: 'main', description: 'Branch (if using inline pipeline)')
    booleanParam(name: 'SKIP_TESTS', defaultValue: true, description: 'Skip tests during build')
    string(name: 'APP_PORT',  defaultValue: '8080', description: 'Spring Boot server.port')
    string(name: 'HEALTH_PATH', defaultValue: '/', description: 'Health path (e.g., / or /actuator/health)')
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timeout(time: 30, unit: 'MINUTES')
    // ansiColor('xterm')  <-- removed because the plugin isn't installed
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        // If this pipeline is from SCM, Jenkins already did the checkout automatically.
        // If you’re pasting this as an inline pipeline job, uncomment:
        // git url: params.REPO_URL, branch: params.BRANCH
      }
    }

    stage('Show Tool Versions') {
      steps {
        bat 'git --version || ver'
        bat 'java -version || ver'
        bat 'mvn -v || ver'
      }
    }

    stage('Build') {
      steps {
        script {
          def skip = params.SKIP_TESTS ? '-DskipTests' : ''
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
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Run Application') {
      steps {
        script {
          // Find jar (single-line PowerShell)
          def jarPath = bat(returnStdout: true, script: """
            powershell -NoProfile -ExecutionPolicy Bypass -Command ^
              "$ErrorActionPreference='SilentlyContinue'; \$j = Get-ChildItem -Path 'target\\*.jar' | Where-Object { \$_.Name -notmatch 'sources|javadoc' } | Sort-Object LastWriteTime -Descending | Select-Object -First 1; if (\$j) { \$j.FullName }"
          """).trim()

          if (!jarPath) {
            error "No runnable JAR found in target/. Ensure spring-boot-maven-plugin repackages the JAR."
          }
          echo "Found JAR: ${jarPath}"

          // Start app in background and capture PID
          bat """
            powershell -NoProfile -ExecutionPolicy Bypass -Command ^
              "$ErrorActionPreference='Stop'; ^
               \$log = Join-Path (Get-Location) 'app.log'; ^
               if (Test-Path \$log) { Remove-Item \$log -Force }; ^
               \$args = @('-jar','${jarPath.replace('\\','/')}','--server.port=${params.APP_PORT}'); ^
               \$p = Start-Process -FilePath 'java' -ArgumentList \$args -PassThru -WindowStyle Hidden -RedirectStandardOutput \$log -RedirectStandardError \$log; ^
               Set-Content -Path app.pid -Value \$p.Id; ^
               'Started PID: ' + \$p.Id | Out-File -FilePath \$log -Append"
          """

          // Wait for health URL to respond
          def healthUrl = "http://localhost:${params.APP_PORT}${params.HEALTH_PATH}"
          def status = bat(returnStatus: true, script: """
            powershell -NoProfile -ExecutionPolicy Bypass -Command ^
              "$ErrorActionPreference='SilentlyContinue'; ^
               \$u='${healthUrl}'; \$deadline=(Get-Date).AddSeconds(90); ^
               while((Get-Date) -lt \$deadline){ ^
                 try { \$r=Invoke-WebRequest -Uri \$u -UseBasicParsing -TimeoutSec 3; if([int]\$r.StatusCode -ge 100){ exit 0 } } catch {}; ^
                 Start-Sleep -Seconds 3 ^
               }; exit 1"
          """)
          if (status != 0) {
            error "App did not respond at ${healthUrl} within 90s. Check app.log."
          }

          echo "App responded at ${healthUrl}"
          bat "powershell -NoProfile -ExecutionPolicy Bypass -Command \"Get-Content .\\app.log -Tail 80 -ErrorAction SilentlyContinue\""
        }
      }
      post {
        always {
          script {
            if (fileExists('app.pid')) {
              def pid = readFile('app.pid').trim()
              echo "Stopping app (PID ${pid}) ..."
              bat "taskkill /PID ${pid} /F || ver"
            } else {
              echo "No app.pid found; nothing to stop."
            }
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'target\\*.jar', fingerprint: true
      echo 'Pipeline finished.'
    }
  }
}
