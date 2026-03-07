pipeline {
  agent any

  parameters {
    // Used only if you run this job as an inline pipeline. For Pipeline-from-SCM we use "checkout scm".
    string(name: 'REPO_URL',   defaultValue: 'https://github.com/NagaKarthikSarma/jenkinstest.git', description: 'Repository URL (if using inline pipeline)')
    string(name: 'BRANCH',     defaultValue: 'main', description: 'Branch (if using inline pipeline)')
    booleanParam(name: 'SKIP_TESTS', defaultValue: true, description: 'Skip tests during build')
    string(name: 'APP_PORT',   defaultValue: '8081', description: 'Spring Boot server.port')
    string(name: 'HEALTH_PATH', defaultValue: '/', description: 'Health path (e.g., / or /actuator/health)')
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timeout(time: 30, unit: 'MINUTES')
    skipDefaultCheckout(true)   // we will checkout explicitly
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        // For Pipeline-from-SCM jobs, this checks out the same repo/branch that contains this Jenkinsfile
        checkout scm

        // If you run as an inline job instead, comment the line above and use:
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
          // 1) Find the runnable JAR (single-line PowerShell so cmd.exe doesn't split pipes)
          def jarPath = bat(returnStdout: true, script: '''
            powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='SilentlyContinue'; $j = Get-ChildItem -Path 'target\\*.jar' | Where-Object { $_.Name -notmatch 'sources|javadoc' } | Sort-Object LastWriteTime -Descending | Select-Object -First 1; if ($j) { $j.FullName }"
          ''').trim()

          if (!jarPath) {
            error "No runnable JAR found in target/. Ensure spring-boot-maven-plugin repackages the JAR."
          }
          echo "Found JAR: ${jarPath}"

          // Normalize path and compose health URL
          def normalizedJar = jarPath.replace('\\','/')
          def healthUrl = "http://localhost:${params.APP_PORT}${params.HEALTH_PATH}"

          // 2) Export values so PowerShell can read them as $env:VAR
          withEnv([
            "JAR_PATH=${normalizedJar}",
            "APP_PORT=${params.APP_PORT}",
            "HEALTH_PATH=${params.HEALTH_PATH}",
            "HEALTH_URL=${healthUrl}"
          ]) {

            // 3) Start app in background, redirect OUT and ERR to different files, capture PID
            bat '''
              powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $logOut = Join-Path (Get-Location) 'app.out.log'; $logErr = Join-Path (Get-Location) 'app.err.log'; if (Test-Path $logOut) { Remove-Item $logOut -Force }; if (Test-Path $logErr) { Remove-Item $logErr -Force }; $args = @('-jar', $env:JAR_PATH, '--server.port=' + $env:APP_PORT); $p = Start-Process -FilePath 'java' -ArgumentList $args -PassThru -WindowStyle Hidden -RedirectStandardOutput $logOut -RedirectStandardError $logErr; Set-Content -Path app.pid -Value $p.Id; 'Started PID: ' + $p.Id | Out-File -FilePath $logOut -Append"
            '''

            // 4) Wait for health URL to respond (any HTTP status means it's responding)
            def status = bat(returnStatus: true, script: '''
              powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='SilentlyContinue'; $u=$env:HEALTH_URL; $deadline=(Get-Date).AddSeconds(90); while((Get-Date) -lt $deadline){ try { $r=Invoke-WebRequest -Uri $u -UseBasicParsing -TimeoutSec 3; if([int]$r.StatusCode -ge 100){ exit 0 } } catch {}; Start-Sleep -Seconds 3 }; exit 1"
            ''')
            if (status != 0) {
              // show whatever logs we have for easier debugging
              bat 'powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-Content .\\app.out.log -Tail 200 -ErrorAction SilentlyContinue; Get-Content .\\app.err.log -Tail 200 -ErrorAction SilentlyContinue"'
              error "App did not respond at ${healthUrl} within 90s. Check app.out.log/app.err.log."
            }

            echo "App responded at ${healthUrl}"
            // Show a tail of both logs
            bat 'powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-Content .\\app.out.log -Tail 80 -ErrorAction SilentlyContinue; Get-Content .\\app.err.log -Tail 80 -ErrorAction SilentlyContinue"'
          }
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
