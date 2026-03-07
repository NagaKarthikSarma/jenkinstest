pipeline {
  agent any

  parameters {
    // If your job is "Pipeline script from SCM", checkout scm is automatic — otherwise this REPO/BRANCH are unused.
    string(name: 'REPO_URL', defaultValue: 'https://github.com/NagaKarthikSarma/jenkinstest.git', description: 'Repository URL (if using inline pipeline)')
    string(name: 'BRANCH',    defaultValue: 'main', description: 'Branch (if using inline pipeline)')
    booleanParam(name: 'SKIP_TESTS', defaultValue: true, description: 'Skip tests during build')
    string(name: 'APP_PORT',  defaultValue: '8080', description: 'Spring Boot server.port')
    string(name: 'HEALTH_PATH', defaultValue: '/', description: 'Health path (e.g., / or /actuator/health)')
  }

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '20'))
    timeout(time: 20, unit: 'MINUTES')
  }

  stages {
    stage('Checkout') {
      steps {
        deleteDir()
        // If this pipeline comes from SCM (Jenkinsfile in repo), 'checkout scm' is not needed here,
        // Jenkins already did "Declarative: Checkout SCM". If you run inline, uncomment the next line:
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
      def jarPath = bat(returnStdout: true, script: """
        powershell -NoProfile -ExecutionPolicy Bypass -Command ^
          "$j = Get-ChildItem -Path 'target\\*.jar' | Where-Object { \$_.Name -notmatch 'sources|javadoc' } | Sort-Object LastWriteTime -Descending | Select-Object -First 1; if ($j) { $j.FullName }"
      """).trim()
      if (!jarPath) { error "No runnable JAR found." }
      bat "start /MIN cmd /c java -jar \"${jarPath}\" --server.port=${params.APP_PORT} > app.log 2>&1"
      bat "powershell -NoProfile -Command \"Start-Sleep -Seconds 8; Get-Content .\\app.log -Tail 80 -ErrorAction SilentlyContinue\""
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
