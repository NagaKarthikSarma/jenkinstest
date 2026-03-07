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
          // 1) Find the runnable JAR
          def jarPath = bat(returnStdout: true, script: '''
            powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='SilentlyContinue'; $j = Get-ChildItem -Path 'target\\*.jar' | Where-Object { $_.Name -notmatch 'sources|javadoc' } | Sort-Object LastWriteTime -Descending | Select-Object -First 1; if ($j) { $j.FullName }"
          ''').trim()

          if (!jarPath) {
            error "No runnable JAR found in target/. Ensure spring-boot-maven-plugin repackages the JAR."
          }
          echo "Found JAR: ${jarPath}"

          def normalizedJar = jarPath.replace('\\','/')
          def healthUrl = "http://localhost:${params.APP_PORT}${params.HEALTH_PATH}"

          withEnv([
            "JAR_PATH=${normalizedJar}",
            "APP_PORT=${params.APP_PORT}",
            "HEALTH_URL=${healthUrl}"
          ]) {

            // 2) FIX: Start process WITHOUT trying to write to the log file simultaneously via Out-File
            bat '''
              powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $logOut = Join-Path (Get-Location) 'app.out.log'; $logErr = Join-Path (Get-Location) 'app.err.log'; if (Test-Path $logOut) { Remove-Item $logOut -Force }; if (Test-Path $logErr) { Remove-Item $logErr -Force }; $args = @('-jar', $env:JAR_PATH, '--server.port=' + $env:APP_PORT); $p = Start-Process -FilePath 'java' -ArgumentList $args -PassThru -WindowStyle Hidden -RedirectStandardOutput $logOut -RedirectStandardError $logErr; Set-Content -Path app.pid -Value $p.Id"
            '''

            // 3) Wait for health URL
            def status = bat(returnStatus: true, script: '''
              powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='SilentlyContinue'; $u=$env:HEALTH_URL; $deadline=(Get-Date).AddSeconds(90); while((Get-Date) -lt $deadline){ try { $r=Invoke-WebRequest -Uri $u -UseBasicParsing -TimeoutSec 3; if([int]$r.StatusCode -ge 100){ exit 0 } } catch {}; Start-Sleep -Seconds 3 }; exit 1"
            ''')
            
            if (status != 0) {
              bat 'powershell -NoProfile -ExecutionPolicy Bypass -Command "Get-Content .\\app.out.log -Tail 50; Get-Content .\\app.err.log -Tail 50"'
              error "App did not respond at ${healthUrl} within 90s."
            }

            echo "App responded at ${healthUrl}"
          }
        }
      }
      // ... keep your post { always { taskkill } } logic as it is
    }
    
  }

  post {
    always {
      archiveArtifacts artifacts: 'target\\*.jar', fingerprint: true
      echo 'Pipeline finished.'
    }
  }
}
