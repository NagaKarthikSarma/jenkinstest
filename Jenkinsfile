pipeline {
    agent any // ensure this runs on a Windows agent

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        // Fixed values (no parameters)
        GIT_URL    = 'https://github.com/NagaKarthikSarma/jenkinstest.git'
        GIT_BRANCH = 'main'

        // App/runtime config
        APP_PORT   = '8081'
        JAVA_EXE   = 'java'
        START_ARGS = '-Xms256m -Xmx512m'
        JAR_GLOB   = 'target\\*.jar'
        HEALTH_URL = "http://localhost:8081/"

        // Paths
        WORKDIR    = "${env.WORKSPACE}"
        LOG_DIR    = "${env.WORKSPACE}\\logs"
        PID_FILE   = "${env.WORKSPACE}\\app.pid"
        OUT_LOG    = "${env.WORKSPACE}\\app.out"
        ERR_LOG    = "${env.WORKSPACE}\\app.err"
    }

    stages {
        stage('Checkout') {
            steps {
                git url: env.GIT_URL, branch: env.GIT_BRANCH
            }
        }

        stage('Build') {
            steps {
                bat 'mvn -version'
                bat 'mvn -B -DskipTests clean package'
            }
        }

        stage('Stop Previous App') {
            steps {
                echo "Attempting to stop previously running app…"

                // Stop via PID file if present (ignore errors)
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

                // Fallback: stop whoever is listening on APP_PORT (always exit 0)
                bat '''
                    powershell -NoProfile -Command "$p = Get-NetTCPConnection -LocalPort $env:APP_PORT -ErrorAction SilentlyContinue | Select-Object -First 1; if($p -and $p.OwningProcess) { try { Stop-Process -Id $p.OwningProcess -Force -ErrorAction Stop; Start-Sleep -Seconds 2 } catch {} }; exit 0"
                '''
            }
        }

        stage('Locate JAR') {
            steps {
                script {
                    def jarPath = null
                    // Try Pipeline Utility Steps if installed; otherwise fallback
                    try {
                        def files = findFiles(glob: env.JAR_GLOB)
                        if (files && files.size() > 0) {
                            jarPath = files[0].path.replace('/', '\\')
                        }
                    } catch (ignored) {}

                    if (!jarPath) {
                        withEnv(["JARGLOB=${env.JAR_GLOB}"]) {
                            bat '''
                                setlocal enabledelayedexpansion
                                set "JARFILE="
                                for /f "delims=" %%F in ('dir /b /a:-d %JARGLOB%') do (
                                    set "JARFILE=%%F"
                                    goto :found
                                )
                                echo NOJAR> jar.tmp
                                exit /b 0
                                :found
                                echo !JARFILE!> jar.tmp
                                endlocal
                            '''
                        }
                        def name = readFile('jar.tmp').trim()
                        if (name == 'NOJAR' || name == '') {
                            error "No JAR found with glob: ${env.JAR_GLOB}"
                        }
                        def baseDir = env.JAR_GLOB.contains('\\') ? env.JAR_GLOB.substring(0, env.JAR_GLOB.lastIndexOf('\\')) : ''
                        jarPath = baseDir ? "${env.WORKSPACE}\\${baseDir}\\${name}" : "${env.WORKSPACE}\\${name}"
                    }

                    env.JAR_PATH = jarPath
                    echo "Resolved JAR: ${env.JAR_PATH}"
                }
            }
        }

        stage('Rotate Logs') {
            steps {
                bat '''
                    if not exist "%LOG_DIR%" mkdir "%LOG_DIR%"
                    powershell -NoProfile -Command ^
                      "$ts = (Get-Date).ToString('yyyyMMdd_HHmmss'); " ^
                      "$srcs = @('$env:OUT_LOG','$env:ERR_LOG'); " ^
                      "foreach($s in $srcs){ if(Test-Path $s){ $name = Split-Path $s -Leaf; $dest = Join-Path '$env:LOG_DIR' ($name -replace '\\.', ('_' + $ts + '.')); Move-Item -Force $s $dest } }"
                '''
            }
        }

        stage('Start App (Detached)') {
            steps {
                echo "Starting app in background (detached)…"
                bat '''
                    set "JARPATH=%JAR_PATH%"
                    if not exist "%JARPATH%" (
                      echo JAR does not exist: %JARPATH%
                      exit /b 2
                    )

                    powershell -NoProfile -Command ^
                      "$jar = [IO.Path]::GetFullPath('$env:JARPATH'); " ^
                      "$wd  = [IO.Path]::GetFullPath('$env:WORKDIR'); " ^
                      "$args = '$env:START_ARGS -Dserver.port=$env:APP_PORT -jar ' + '\"' + $jar + '\"'; " ^
                      "$p = Start-Process -FilePath '$env:JAVA_EXE' -ArgumentList $args -WorkingDirectory $wd -RedirectStandardOutput '$env:OUT_LOG' -RedirectStandardError '$env:ERR_LOG' -WindowStyle Hidden -PassThru; " ^
                      "$p.Id | Set-Content -Encoding Ascii '$env:PID_FILE'"

                    echo Started. PID saved in %PID_FILE%
                '''
            }
        }

        stage('Health Check') {
            steps {
                echo "Probing health: ${env.HEALTH_URL}"
                bat '''
                    powershell -NoProfile -Command ^
                      "$u='$env:HEALTH_URL'; " ^
                      "for($i=0; $i -lt 30; $i++){ " ^
                      "  try { $r = Invoke-WebRequest -Uri $u -UseBasicParsing -TimeoutSec 5; " ^
                      "        if( ($r.StatusCode -ge 200 -and $r.StatusCode -lt 400) -and ($r.Content -match 'UP' -or $r.StatusCode -eq 200) ) { exit 0 } " ^
                      "      } catch { } " ^
                      "  Start-Sleep -Seconds 2 " ^
                      "} " ^
                      "exit 1"
                '''
            }
        }
    }

    post {
        success {
            echo "Lifecycle completed successfully."
            archiveArtifacts artifacts: 'logs\\*.out, logs\\*.err, app.out, app.err', allowEmptyArchive: true, fingerprint: true
        }
        failure {
            echo "Lifecycle failed."
            archiveArtifacts artifacts: 'logs\\*.out, logs\\*.err, app.out, app.err', allowEmptyArchive: true, fingerprint: true
        }
        unstable {
            echo "Lifecycle finished UNSTABLE."
            archiveArtifacts artifacts: 'logs\\*.out, logs\\*.err, app.out, app.err', allowEmptyArchive: true, fingerprint: true
        }
        aborted {
            echo "Lifecycle ABORTED."
            archiveArtifacts artifacts: 'logs\\*.out, logs\\*.err, app.out, app.err', allowEmptyArchive: true, fingerprint: true
        }
        always {
            echo "PID file: ${env.PID_FILE}"
            echo "Logs: ${env.OUT_LOG}, ${env.ERR_LOG}, and rotated files under ${env.LOG_DIR}"
        }
    }
}
