pipeline {
    agent any / <-- Change to your Windows agent label if different

    options {
        timestamps()
        ansiColor('xterm')
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()  // avoid overlapping lifecycle runs
        timeout(time: 30, unit: 'MINUTES')
    }

    parameters {
        choice(name: 'ACTION',
               choices: ['BUILD_AND_RESTART', 'RESTART', 'START', 'STOP', 'BUILD_ONLY'],
               description: 'Lifecycle action to perform')

        string(name: 'GIT_URL', defaultValue: 'https://github.com/NagaKarthikSarma/jenkinstest.git',
               description: 'Repository URL')
        string(name: 'GIT_BRANCH', defaultValue: 'main', description: 'Branch to build')

        string(name: 'JAR_GLOB', defaultValue: 'target\\*.jar',
               description: 'Glob to locate the built JAR (Windows-style backslashes)')
        string(name: 'JAVA_EXE', defaultValue: 'java',
               description: 'Path to java.exe or just "java" if on PATH')

        string(name: 'APP_PORT', defaultValue: '8080', description: 'Port your app listens on')
        string(name: 'START_ARGS', defaultValue: '-Xms256m -Xmx512m',
               description: 'Extra JVM args (do NOT include -jar here)')

        string(name: 'HEALTH_PATH', defaultValue: '/actuator/health',
               description: 'Relative health path (set to "/" if not using Actuator)')

        string(name: 'WORKING_DIR', defaultValue: '',
               description: 'Working directory for app (leave empty to use workspace)')
    }

    environment {
        // Derived env
        WORKDIR      = "${params.WORKING_DIR ?: env.WORKSPACE}"
        LOG_DIR      = "${env.WORKSPACE}\\logs"
        PID_FILE     = "${env.WORKSPACE}\\app.pid"
        HEALTH_URL   = "http://localhost:${params.APP_PORT}${params.HEALTH_PATH}"
        // Where stdout/stderr go (rotated on every start)
        OUT_LOG      = "${env.WORKSPACE}\\app.out"
        ERR_LOG      = "${env.WORKSPACE}\\app.err"
    }

    stages {
        stage('Checkout') {
            when {
                expression { return params.ACTION in ['BUILD_AND_RESTART', 'BUILD_ONLY'] }
            }
            steps {
                git url: params.GIT_URL, branch: params.GIT_BRANCH
            }
        }

        stage('Build') {
            when {
                expression { return params.ACTION in ['BUILD_AND_RESTART', 'BUILD_ONLY'] }
            }
            steps {
                // Configure tools in Manage Jenkins > Global Tool Configuration and uncomment if desired:
                // tools { maven 'Maven 3.x'; jdk 'JDK 17' }
                bat 'mvn -version'
                bat 'mvn -B -DskipTests clean package'
            }
        }

        stage('Stop Previous App') {
            when {
                expression { return params.ACTION in ['BUILD_AND_RESTART', 'RESTART', 'STOP'] }
            }
            steps {
                echo "Attempting to stop previously running app…"
                // 1) Stop by known PID (from previous run)
                bat '''
                    setlocal
                    if exist "%PID_FILE%" (
                      for /f "usebackq delims=" %%P in ("%PID_FILE%") do (
                        echo Stopping PID from PID_FILE: %%P
                        powershell -NoProfile -Command "try { Stop-Process -Id %%P -Force -ErrorAction Stop } catch {}"
                      )
                      del /q "%PID_FILE%" 2>nul
                    )
                    endlocal
                '''
                // 2) Best-effort stop by listening port (if PID file was missing)
                bat """
                    powershell -NoProfile -Command ^
                      "$p = Get-NetTCPConnection -LocalPort ${params.APP_PORT} -ErrorAction SilentlyContinue | Select-Object -First 1; " ^
                      "if(\$p -and \$p.OwningProcess) { " ^
                      "  try { Stop-Process -Id \$p.OwningProcess -Force -ErrorAction Stop; Start-Sleep -Seconds 2 } catch {} " ^
                      "} "
                """
            }
        }

        stage('Locate JAR') {
            when {
                expression { return params.ACTION in ['BUILD_AND_RESTART', 'RESTART', 'START'] }
            }
            steps {
                script {
                    // Try plugin first if installed; fallback to cmd
                    def jarPath = null
                    try {
                        def files = findFiles(glob: params.JAR_GLOB)
                        if (files && files.size() > 0) {
                            jarPath = files[0].path.replace('/', '\\')
                        }
                    } catch (ignored) {}

                    if (!jarPath) {
                        // Fallback: find the first JAR using dir /b
                        // Note: assumes JAR resides in "target" as per default Maven layout
                        bat """
                            setlocal enabledelayedexpansion
                            set "JARFILE="
                            for /f "delims=" %%F in ('dir /b /a:-d ${params.JAR_GLOB}') do (
                                set "JARFILE=%%F"
                                goto :found
                            )
                            echo NOJAR> jar.tmp
                            exit /b 0
                            :found
                            echo !JARFILE!> jar.tmp
                            endlocal
                        """
                        def name = readFile('jar.tmp').trim()
                        if (name == 'NOJAR' || name == '') {
                            error "No JAR found with glob: ${params.JAR_GLOB}"
                        }
                        // Compute full path (relative to workspace)
                        // If your glob is 'target\\*.jar', jar resides in '%WORKSPACE%\\target\\<name>'
                        def baseDir = params.JAR_GLOB.contains('\\') ? params.JAR_GLOB.substring(0, params.JAR_GLOB.lastIndexOf('\\')) : ''
                        jarPath = baseDir ? "${env.WORKSPACE}\\${baseDir}\\${name}" : "${env.WORKSPACE}\\${name}"
                    }

                    env.JAR_PATH = jarPath
                    echo "Resolved JAR: ${env.JAR_PATH}"
                }
            }
        }

        stage('Rotate Logs') {
            when {
                expression { return params.ACTION in ['BUILD_AND_RESTART', 'RESTART', 'START'] }
            }
            steps {
                bat """
                    if not exist "%LOG_DIR%" mkdir "%LOG_DIR%"
                    powershell -NoProfile -Command ^
                      "$ts = (Get-Date).ToString('yyyyMMdd_HHmmss'); " ^
                      "$srcs = @('%OUT_LOG%','%ERR_LOG%'); " ^
                      "foreach(\$s in \$srcs){ if(Test-Path \$s){ \$name = Split-Path \$s -Leaf; \$dest = Join-Path '%LOG_DIR%' (\$name -replace '\\.', ('_' + \$ts + '.')); Move-Item -Force \$s \$dest } }"
                """
            }
        }

        stage('Start App (Detached)') {
            when {
                expression { return params.ACTION in ['BUILD_AND_RESTART', 'RESTART', 'START'] }
            }
            steps {
                echo "Starting app in background (detached)…"
                bat """
                    set "JARPATH=%JAR_PATH%"
                    if not exist "%JARPATH%" (
                      echo JAR does not exist: %JARPATH%
                      exit /b 2
                    )

                    powershell -NoProfile -Command ^
                      "$jar = [IO.Path]::GetFullPath('%JARPATH%'); " ^
                      "$wd  = [IO.Path]::GetFullPath('%WORKDIR%'); " ^
                      "$args = '%START_ARGS% -Dserver.port=${params.APP_PORT} -jar ' + '\"' + $jar + '\"'; " ^
                      "$p = Start-Process -FilePath '%JAVA_EXE%' -ArgumentList $args -WorkingDirectory $wd -RedirectStandardOutput '%OUT_LOG%' -RedirectStandardError '%ERR_LOG%' -WindowStyle Hidden -PassThru; " ^
                      "$p.Id | Set-Content -Encoding Ascii '%PID_FILE%'"

                    echo Started. PID saved in %PID_FILE%
                """
            }
        }

        stage('Health Check') {
            when {
                expression { return params.ACTION in ['BUILD_AND_RESTART', 'RESTART', 'START'] }
            }
            steps {
                echo "Probing health: ${HEALTH_URL}"
                // Try up to 30 times (≈ 60s) for UP or HTTP 200-399
                bat """
                    powershell -NoProfile -Command ^
                      "$u='${HEALTH_URL}'; " ^
                      "for(\$i=0; \$i -lt 30; \$i++){ " ^
                      "  try { " ^
                      "    \$r = Invoke-WebRequest -Uri \$u -UseBasicParsing -TimeoutSec 5; " ^
                      "    if( (\$r.StatusCode -ge 200 -and \$r.StatusCode -lt 400) -and (\$r.Content -match 'UP' -or \$r.StatusCode -eq 200) ) { exit 0 } " ^
                      "  } catch { } " ^
                      "  Start-Sleep -Seconds 2 " ^
                      "} " ^
                      "exit 1"
                """
            }
        }
    }

    post {
        success {
            echo "Action '${params.ACTION}' completed successfully."
            archiveArtifacts artifacts: 'logs\\*.out, logs\\*.err, app.out, app.err', allowEmptyArchive: true, fingerprint: true
        }
        unsuccessful {
            echo "Action '${params.ACTION}' did not complete successfully."
            archiveArtifacts artifacts: 'logs\\*.out, logs\\*.err, app.out, app.err', allowEmptyArchive: true, fingerprint: true
        }
        always {
            // Print where to find logs & PID
            echo "PID file: ${env.PID_FILE}"
            echo "Logs: ${env.OUT_LOG}, ${env.ERR_LOG}, and rotated files under ${env.LOG_DIR}"
        }
    }
}
