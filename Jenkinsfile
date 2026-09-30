import groovy.json.JsonOutput
@Library('internal-jenkins-lib') _

ENV_PARAMETERS = env_parameters AWS_REGION, ENVIRONMENT

S3_BUCKET_BY_ENV_REGION = [
    'prod-us-east-1': 'appname-prod-us-east-1',
    'qa-us-east-1':   'appname-qa-us-east-1',
    'uat-us-east-1':  'appname-uat-us-east-1',
    'prod-us-west-2': 'appname-prod-us-west-2',
    'qa-us-west-2':   'appname-qa-us-west-2',
    'uat-us-west-2':  'appname-uat-us-west-2'
]

def runSsmCommandInline(instanceId, command, timeoutSeconds = 60, echoOutput = true) {
    def commandsJson = JsonOutput.toJson(['commands': [command]])
    writeFile file: 'ssm_payload.json', text: commandsJson

    def responseText = sh(
        script: """
            aws ssm send-command \
                --timeout-seconds ${timeoutSeconds} \
                --instance-ids ${instanceId} \
                --document-name AWS-RunPowerShellScript \
                --parameters file://ssm_payload.json
        """,
        returnStdout: true
    ).trim()

    def responseJson = readJSON text: responseText
    def commandId = responseJson['Command']['CommandId']
    def status = responseJson['Command']['Status']

    while (status == 'Pending' || status == 'InProgress') {
        sleep(5)
        responseText = sh(
            script: """
                aws ssm get-command-invocation \
                    --command-id ${commandId} \
                    --instance-id ${instanceId}
            """,
            returnStdout: true
        ).trim()

        responseJson = readJSON text: responseText
        status = responseJson['Status']
    }

    if (echoOutput) {
        echo "SSM STDOUT: ${responseJson['StandardOutputContent']}"
    } else {
        echo "SSM STDOUT: [suppressed - sensitive data]"
    }
    if (responseJson['StandardErrorContent']) {
        echo "SSM STDERR: ${responseJson['StandardErrorContent']}"
    }

    if (status != 'Success') {
        error("SSM command failed (Status = ${status}). ${responseJson['StandardErrorContent']}")
    }

    return responseJson['StandardOutputContent']
}

def getDeployedClientNames(instanceId) {
    def output = runSsmCommandInline(
        instanceId,
        "Import-Module WebAdministration; Get-ChildItem IIS:\\AppPools | Select-Object -ExpandProperty Name | Where-Object { \$_ -notin @('.NET v4.5', '.NET v4.5 Classic', 'DefaultAppPool') }"
    )
    return output.readLines().collect { it.trim() }.findAll { it }
}

@NonCPS
def parseConfigConnString(String content) {
    def matcher = content =~ /name=['"]database['"][^>]*connectionString=['"]([^'"]+)['"]/
    return matcher ? matcher[0][1] : null
}

@NonCPS
def parseDbHost(String connString) {
    def matcher = connString =~ /Host=([^;]+)/
    return matcher ? matcher[0][1] : null
}

@NonCPS
def parseDbName(String connString) {
    def matcher = connString =~ /Database=([^;]+)/
    return matcher ? matcher[0][1] : null
}

@NonCPS
def parseVersionResult(String output) {
    def matcher = output =~ /VERSION_RESULT:([^|]+)\|(.+)/
    return matcher ? [matcher[0][1].trim(), matcher[0][2].trim()] : null
}

@NonCPS
String toBase64(String content) {
    def bytes = content.getBytes('UTF-8')
    def table = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
    def sb = new StringBuilder()
    int i = 0
    while (i < bytes.length) {
        int b0 = bytes[i] & 0xFF
        int b1 = (i + 1 < bytes.length) ? (bytes[i + 1] & 0xFF) : 0
        int b2 = (i + 2 < bytes.length) ? (bytes[i + 2] & 0xFF) : 0

        sb.append(table[(b0 >> 2) & 0x3F])
        sb.append(table[((b0 << 4) | (b1 >> 4)) & 0x3F])
        sb.append(i + 1 < bytes.length ? table[((b1 << 2) | (b2 >> 6)) & 0x3F] : '=')
        sb.append(i + 2 < bytes.length ? table[b2 & 0x3F] : '=')

        i += 3
    }
    return sb.toString()
}

pipeline {
    agent any
    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(daysToKeepStr: '30'))
    }
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['prod', 'qa', 'uat'], description: 'Environment identifier')
        choice(name: 'AWS_REGION', choices: ['us-east-1','us-west-2'], description: 'AWS Region to deploy the stack')
        string(name: 'ENV_INSTANCE', defaultValue: '', description: 'Environment instance identifier, e.g. 1, 2, 3')
        string(name: 'CLIENT_NAME', defaultValue: '', description: "Must match the customer's deployment name on the target stack")
        booleanParam(name: 'SKIP_BACKUP', defaultValue: false, description: 'Check to skip client folder zip and database backup (config capture still runs). Not recommended for prod.')
        booleanParam(name: 'SKIP_BASE_TENANT_UPDATE', defaultValue: false, description: 'Check to use the existing base tenant version for a customer version upgrade only.')
        string(name: 'APP_BRANCH', defaultValue: 'Develop', description: 'Enter the app build branch, e.g. Develop, ReleaseCandidate, Release')
        string(name: 'REPO_BRANCH', defaultValue: 'master', description: 'Name of git branch to use')
    }

    stages {
        stage('Client Name Validation') {
            when {
                expression { return CLIENT_NAME?.trim() }
            }
            steps {
                script {
                    withAWS(role: "${ENV_PARAMETERS['DeploymentRole']}", roleAccount: "${ENV_PARAMETERS['AwsAccount']}", duration: 10800, region: "${AWS_REGION}") {
                        app_server_instance_id = sh (
                            script: "aws cloudformation describe-stack-resources --stack-name ${ENVIRONMENT}-${ENV_INSTANCE} --logical-resource-id AppServerInstance0 --query StackResources[0].PhysicalResourceId",
                            returnStdout: true
                        ).trim()

                        echo "Checking deployed clients on ${ENVIRONMENT}-${ENV_INSTANCE}..."
                        def deployedClients = getDeployedClientNames(app_server_instance_id)

                        if (!deployedClients.contains(CLIENT_NAME)) {
                            error("CLIENT_NAME validation FAILED: '${CLIENT_NAME}' was not found on ${ENVIRONMENT}-${ENV_INSTANCE}. Valid deployed clients: ${deployedClients.join(', ')}. Please check the environment and confirm the deployment name before proceeding.")
                        }
                        echo "CLIENT_NAME validated successfully: '${CLIENT_NAME}' matches a deployed client on ${ENVIRONMENT}-${ENV_INSTANCE}."

                        echo "Stopping app pool '${CLIENT_NAME}'..."
                        runSsmCommandInline(
                            app_server_instance_id,
                            "\$CLIENT_NAME = '${CLIENT_NAME}'; if ((Get-WebAppPoolState -Name \$CLIENT_NAME).Value -ne 'Stopped') { Stop-WebAppPool -Name \$CLIENT_NAME }; \$timeoutSec = 30; \$elapsed = 0; while ((Get-WebAppPoolState -Name \$CLIENT_NAME).Value -ne 'Stopped' -and \$elapsed -lt \$timeoutSec) { Start-Sleep -Seconds 1; \$elapsed++ }; if ((Get-WebAppPoolState -Name \$CLIENT_NAME).Value -ne 'Stopped') { throw \"App pool did not stop within \$timeoutSec seconds\" } else { Write-Output 'App pool stopped successfully' }"
                        )
                        echo "App pool '${CLIENT_NAME}' stopped successfully."
                    }
                }
            }
        }
        stage('Backing up Client Data') {
            when {
                expression { return CLIENT_NAME?.trim() }
            }
            steps {
                script {
                    withAWS(role: "${ENV_PARAMETERS['DeploymentRole']}", roleAccount: "${ENV_PARAMETERS['AwsAccount']}", duration: 10800, region: "${AWS_REGION}") {
                        app_server_instance_id = sh (
                            script: "aws cloudformation describe-stack-resources --stack-name ${ENVIRONMENT}-${ENV_INSTANCE} --logical-resource-id AppServerInstance0 --query StackResources[0].PhysicalResourceId",
                            returnStdout: true
                        ).trim()

                        // --- Capture the 4 config files' content, parse DB connection details ---
                        echo "Capturing configuration files for '${CLIENT_NAME}'..."
                        def configCaptureOutput = runSsmCommandInline(
                            app_server_instance_id,
                            "\$CLIENT_NAME = '${CLIENT_NAME}'; \$appsRoot = 'E:\\apps'; \$sourcePath = Join-Path \$appsRoot \$CLIENT_NAME; \$configFiles = @('Services\\CustomerAppSettings.config', 'Services\\CustomerConnectionStrings.config', 'Admin\\CustomerAppSettings.config', 'Admin\\CustomerConnectionStrings.config'); \$capturedContent = @{}; foreach (\$relativePath in \$configFiles) { \$sourceFile = Join-Path \$sourcePath \$relativePath; if (Test-Path \$sourceFile) { \$content = Get-Content \$sourceFile -Raw; \$capturedContent[\$relativePath] = \$content; \$byteCount = [System.Text.Encoding]::UTF8.GetByteCount(\$content); Write-Output \"CONFIG_START:\$relativePath|\$byteCount\"; Write-Output \$content; Write-Output \"CONFIG_END:\$relativePath\" } else { throw \"Config file not found: \$sourceFile\" } }",
                            120,
                            false
                        )

                        def capturedFiles = [:]
                        def lines = configCaptureOutput.readLines()
                        def currentKey = null
                        def buffer = new StringBuilder()
                        for (line in lines) {
                            if (line.startsWith('CONFIG_START:')) {
                                def parts = line.substring('CONFIG_START:'.length()).split('\\|')
                                currentKey = parts[0]
                                buffer = new StringBuilder()
                            } else if (line.startsWith('CONFIG_END:')) {
                                capturedFiles[currentKey] = buffer.toString()
                                echo "Captured: ${currentKey} (${buffer.toString().bytes.length} bytes)"
                                currentKey = null
                            } else if (currentKey != null) {
                                buffer.append(line).append('\n')
                            }
                        }

                        def connStringContent = capturedFiles['Services\\CustomerConnectionStrings.config']
                        def configConnString = parseConfigConnString(connStringContent)
                        if (!configConnString) {
                            error("Could not find the 'database' connection string entry in connectionstrings config")
                        }
                        DB_HOST = parseDbHost(configConnString)
                        DB_NAME = parseDbName(configConnString)
                        if (!DB_HOST || !DB_NAME) {
                            error("Could not parse Host/Database from config connection string")
                        }
                        CONFIG_FILES_CONTENT = capturedFiles

                        echo "Config database identified successfully"

                        if (params.SKIP_BACKUP) {
                            echo "SKIP_BACKUP is checked - skipping client folder zip and database backup."
                        } else {
                            // --- Read source version info from version.json (identifies what's being backed up) ---
                            echo "Reading source version info for '${CLIENT_NAME}'..."
                            def versionOutput = runSsmCommandInline(
                                app_server_instance_id,
                                "\$CLIENT_NAME = '${CLIENT_NAME}'; \$appsRoot = 'E:\\apps'; \$sourcePath = Join-Path \$appsRoot \$CLIENT_NAME; \$versionJsonPath = Join-Path \$sourcePath 'version.json'; if (-not (Test-Path \$versionJsonPath)) { throw \"version.json not found at \$versionJsonPath\" }; \$versionData = Get-Content \$versionJsonPath -Raw | ConvertFrom-Json; \$displayVersion = \$versionData.DisplayVersion; \$internalVersion = \$versionData.InternalVersion; if (-not \$displayVersion -or -not \$internalVersion) { throw \"DisplayVersion/InternalVersion not found in \$versionJsonPath\" }; \$sanitizedDisplay = \$displayVersion -replace '[^a-zA-Z0-9]', '_'; \$sanitizedInternal = \$internalVersion -replace '[^a-zA-Z0-9]', '_'; Write-Output \"VERSION_RESULT:\$sanitizedDisplay|\$sanitizedInternal\"",
                                60
                            )
                            def versionParts = parseVersionResult(versionOutput)
                            if (!versionParts) {
                                error("Could not parse DisplayVersion/InternalVersion from version.json output")
                            }
                            def sanitizedDisplayVersion = versionParts[0]
                            def sanitizedInternalVersion = versionParts[1]
                            def versionFolder = "${sanitizedDisplayVersion}_${sanitizedInternalVersion}"
                            echo "Source version identified: ${sanitizedDisplayVersion} / ${sanitizedInternalVersion}"

                            def bucketKey = "${ENVIRONMENT}-${AWS_REGION}"
                            def s3Bucket = S3_BUCKET_BY_ENV_REGION[bucketKey]
                            if (!s3Bucket) {
                                error("No S3 backup bucket configured for ${ENVIRONMENT} in ${AWS_REGION}. Valid combinations: ${S3_BUCKET_BY_ENV_REGION.keySet().join(', ')}")
                            }
                            def s3BackupUri = "s3://${s3Bucket}/data-backup/${CLIENT_NAME}/${versionFolder}"

                            // --- Zip the tenant folder (local, then upload the same local file to S3 with validated cleanup) ---
                            echo "Backing up client folder for '${CLIENT_NAME}'..."
                            def zipOutput = runSsmCommandInline(
                                app_server_instance_id,
                                "\$CLIENT_NAME = '${CLIENT_NAME}'; \$appsRoot = 'E:\\apps'; \$backupRoot = 'E:\\backups'; \$sourcePath = Join-Path \$appsRoot \$CLIENT_NAME; if (-not (Test-Path \$backupRoot)) { New-Item -ItemType Directory -Path \$backupRoot -Force | Out-Null }; \$dateStamp = Get-Date -Format 'yyyyMMdd'; \$zipFileName = \"\${CLIENT_NAME}_${sanitizedDisplayVersion}_${sanitizedInternalVersion}_\${dateStamp}.zip\"; \$zipFullPath = Join-Path \$backupRoot \$zipFileName; \$startTime = Get-Date; Compress-Archive -Path \$sourcePath -DestinationPath \$zipFullPath -Force; \$elapsed = (Get-Date) - \$startTime; \$zipSize = (Get-Item \$zipFullPath).Length; if (-not (Test-Path \$zipFullPath)) { throw 'Client folder zip FAILED' }; Write-Output \"Backup created: \$zipFullPath (took \$(\$elapsed.ToString()))\"; Write-Output \"ZIP_RESULT:\$zipFileName|\$zipSize\"; try { \$s3ZipTarget = \"${s3BackupUri}/\$zipFileName\"; aws s3 cp \$zipFullPath \$s3ZipTarget; if (\$LASTEXITCODE -ne 0) { throw \"aws s3 cp exited with code \$LASTEXITCODE\" }; Write-Output \"S3_ZIP_RESULT:\$s3ZipTarget\"; \$s3ZipKey = \"data-backup/${CLIENT_NAME}/${versionFolder}/\$zipFileName\"; \$remoteZipSize = (aws s3api head-object --bucket ${s3Bucket} --key \$s3ZipKey --query ContentLength --output text 2>\$null); if (\$LASTEXITCODE -eq 0 -and \$remoteZipSize -eq \$zipSize) { Remove-Item \$zipFullPath -Force; Write-Output \"S3_ZIP_VALIDATED: local zip removed, upload confirmed (\$remoteZipSize bytes)\" } else { Write-Output \"S3_ZIP_VALIDATION_WARNING: could not validate S3 zip upload (local \$zipSize bytes, remote reported '\$remoteZipSize') - local backup retained\" } } catch { Write-Output \"S3_ZIP_UPLOAD_WARNING: S3 zip upload failed, local backup still valid. Error: \$(\$_.Exception.Message)\" }",
                                900
                            )
                            echo zipOutput

                            // --- Lookup processing server, run pg_dump ---
                            echo "Looking up processing server for ${ENVIRONMENT}-${ENV_INSTANCE}..."
                            processing_instance_id = sh (
                                script: "aws cloudformation describe-stack-resources --stack-name ${ENVIRONMENT}-${ENV_INSTANCE} --logical-resource-id ProcessingInstance --query StackResources[0].PhysicalResourceId",
                                returnStdout: true
                            ).trim()

                            // --- Run database backup (detects PG17/16/15 client tools), then upload the same local file to S3 with validated cleanup ---
                            echo "Running database backup for '${CLIENT_NAME}'..."
                            def dbBackupOutput = runSsmCommandInline(
                                processing_instance_id,
                                "\$AWS_REGION = '${AWS_REGION}'; \$dbHost = '${DB_HOST}'; \$dbName = '${DB_NAME}'; \$pgVersions = @('17','16','15','14'); \$psqlExe = \$null; \$pgDumpExe = \$null; foreach (\$v in \$pgVersions) { \$candidate = \"C:\\Program Files\\PostgreSQL\\\$v\\bin\\psql.exe\"; if (Test-Path \$candidate) { \$psqlExe = \$candidate; \$pgDumpExe = \"C:\\Program Files\\PostgreSQL\\\$v\\bin\\pg_dump.exe\"; Write-Output \"PG_VERSION_FOUND: \$v\"; break } }; if (-not \$psqlExe) { throw \"No supported PostgreSQL client tools found (checked versions: \$(\$pgVersions -join ', ')) on processing server. Install one of these versions or contact the pipeline owner.\" }; \$databaseSecretArn = (aws rds describe-db-instances --db-instance-identifier ${ENVIRONMENT}-${ENV_INSTANCE}-database --query 'DBInstances[0].MasterUserSecret.SecretArn' --output text --region \$AWS_REGION); \$databaseAdminPassword = (aws secretsmanager get-secret-value --secret-id \$databaseSecretArn --query SecretString --output text --region \$AWS_REGION | ConvertFrom-Json).password; \$env:PGPASSWORD = \$databaseAdminPassword; \$backupRoot = 'E:\\backups'; if (-not (Test-Path \$backupRoot)) { New-Item -ItemType Directory -Path \$backupRoot -Force | Out-Null }; \$dateStamp = Get-Date -Format 'yyyyMMdd'; \$dbBackupFileName = \"${CLIENT_NAME}_pgdump_${sanitizedDisplayVersion}_${sanitizedInternalVersion}_\${dateStamp}.sql\"; \$dbBackupPath = Join-Path \$backupRoot \$dbBackupFileName; \$startTime = Get-Date; & \$pgDumpExe -h \$dbHost -U db_admin_user -d \$dbName -f \$dbBackupPath; \$elapsed = (Get-Date) - \$startTime; \$dbSize = (Get-Item \$dbBackupPath).Length; if (-not (Test-Path \$dbBackupPath)) { throw 'DB backup FAILED' }; Write-Output \"DB backup created (took \$(\$elapsed.ToString()))\"; Write-Output \"DB_RESULT:\$dbBackupFileName|\$dbSize\"; try { \$s3DbTarget = \"${s3BackupUri}/\$dbBackupFileName\"; aws s3 cp \$dbBackupPath \$s3DbTarget; if (\$LASTEXITCODE -ne 0) { throw \"aws s3 cp exited with code \$LASTEXITCODE\" }; Write-Output \"S3_RESULT:\$s3DbTarget\"; \$s3DbKey = \"data-backup/${CLIENT_NAME}/${versionFolder}/\$dbBackupFileName\"; \$remoteDbSize = (aws s3api head-object --bucket ${s3Bucket} --key \$s3DbKey --query ContentLength --output text 2>\$null); if (\$LASTEXITCODE -eq 0 -and \$remoteDbSize -eq \$dbSize) { Remove-Item \$dbBackupPath -Force; Write-Output \"S3_DB_VALIDATED: local db backup removed, upload confirmed (\$remoteDbSize bytes)\" } else { Write-Output \"S3_DB_VALIDATION_WARNING: could not validate S3 db backup upload (local \$dbSize bytes, remote reported '\$remoteDbSize') - local backup retained\" } } catch { Write-Output \"S3_UPLOAD_WARNING: S3 backup upload failed, local backup still valid. Error: \$(\$_.Exception.Message)\" }; Remove-Item Env:\\PGPASSWORD -ErrorAction SilentlyContinue",
                                300
                            )
                            echo dbBackupOutput
                        }
                    }
                }
            }
        }
        stage('Install latest App update') {
            steps {
                script {
                    currentBuild.displayName = "#${BUILD_NUMBER}-${ENVIRONMENT}-${ENV_INSTANCE}"

                    if (CLIENT_NAME?.trim() && params.SKIP_BASE_TENANT_UPDATE) {
                        echo "SKIP_BASE_TENANT_UPDATE is checked - skipping base tenant install. Client folder will be rebuilt from the current base tenant contents."
                    } else {
                        withAWS(role: "${ENV_PARAMETERS['DeploymentRole']}", roleAccount: "${ENV_PARAMETERS['AwsAccount']}", duration: 10800, region: "${AWS_REGION}") {
                            app_server_instance_id = sh (
                                script: "aws cloudformation describe-stack-resources --stack-name ${ENVIRONMENT}-${ENV_INSTANCE} --logical-resource-id AppServerInstance0 --query StackResources[0].PhysicalResourceId",
                                returnStdout: true
                            ).trim()

                            echo "Stopping base tenant app pool 'AppName Web App Pool'..."
                            runSsmCommandInline(
                                app_server_instance_id,
                                "\$PoolName = 'AppName Web App Pool'; if ((Get-WebAppPoolState -Name \$PoolName).Value -ne 'Stopped') { Stop-WebAppPool -Name \$PoolName }; \$timeoutSec = 30; \$elapsed = 0; while ((Get-WebAppPoolState -Name \$PoolName).Value -ne 'Stopped' -and \$elapsed -lt \$timeoutSec) { Start-Sleep -Seconds 1; \$elapsed++ }; if ((Get-WebAppPoolState -Name \$PoolName).Value -ne 'Stopped') { throw \"Base tenant app pool did not stop within \$timeoutSec seconds\" } else { Write-Output 'Base tenant app pool stopped successfully' }"
                            )
                            echo "Base tenant app pool stopped successfully."

                            echo "Clearing base tenant bin folder..."
                            runSsmCommandInline(
                                app_server_instance_id,
                                "\$binPath = 'E:\\apps\\appname-base-tenant\\Services\\bin'; if (Test-Path \$binPath) { Get-ChildItem -Path \$binPath -Force | Remove-Item -Recurse -Force; Write-Output 'Base tenant bin folder cleared' } else { throw \"Bin folder not found: \$binPath\" }"
                            )
                            echo "Base tenant bin folder cleared successfully."

                            response_text = sh(
                                script: """
                                    aws ssm send-command \
                                        --timeout-seconds 60 \
                                        --instance-ids ${app_server_instance_id} \
                                        --document-name AWS-RunPowerShellScript \
                                        --parameters commands=["E:\\scripts\\Install-Application.ps1 -Update -Environment ${ENVIRONMENT} -EnvInstance ${ENV_INSTANCE} -BranchName ${APP_BRANCH} -AwsRegion ${AWS_REGION}"]
                                        """,
                                returnStdout: true
                            ).trim()

                            println response_text
                            response_json = readJSON text: "${response_text}"
                            command_id = response_json['Command']['CommandId']
                            status = response_json['Command']['Status']

                            while(status.equals("Pending") || status.equals("InProgress")) {
                                sleep (5)
                                response_text = sh(
                                    script: """
                                        aws ssm get-command-invocation \
                                            --command-id ${command_id} \
                                            --instance-id ${app_server_instance_id}""",
                                    returnStdout: true
                                ).trim()

                                println response_text
                                response_json = readJSON text: "${response_text}"
                                status = response_json['Status']
                            }

                            if(response_json['StandardErrorContent']) {
                                error("Powershell script failed")
                            }
                        }
                    }
                }
            }
        }
        stage('Rebuild Client Folder and Configuration') {
            when {
                expression { return CLIENT_NAME?.trim() }
            }
            steps {
                script {
                    withAWS(role: "${ENV_PARAMETERS['DeploymentRole']}", roleAccount: "${ENV_PARAMETERS['AwsAccount']}", duration: 10800, region: "${AWS_REGION}") {
                        app_server_instance_id = sh (
                            script: "aws cloudformation describe-stack-resources --stack-name ${ENVIRONMENT}-${ENV_INSTANCE} --logical-resource-id AppServerInstance0 --query StackResources[0].PhysicalResourceId",
                            returnStdout: true
                        ).trim()

                        // --- Wipe client folder, copy base tenant contents ---
                        echo "Wiping and repopulating client folder for '${CLIENT_NAME}'..."
                        def restoreOutput = runSsmCommandInline(
                            app_server_instance_id,
                            "\$CLIENT_NAME = '${CLIENT_NAME}'; \$appsRoot = 'E:\\apps'; \$sourcePath = Join-Path \$appsRoot \$CLIENT_NAME; \$baseTenantPath = Join-Path \$appsRoot 'appname-base-tenant'; Get-ChildItem -Path \$sourcePath -Force | Remove-Item -Recurse -Force; Copy-Item -Path (Join-Path \$baseTenantPath '*') -Destination \$sourcePath -Recurse -Force; Write-Output 'Client folder wiped and repopulated'",
                            900
                        )
                        echo restoreOutput

                        // --- Write the 4 preserved config files back (base64 to avoid escaping issues) ---
                        echo "Restoring preserved config files for '${CLIENT_NAME}'..."
                        CONFIG_FILES_CONTENT.each { relativePath, content ->
                            def encoded = toBase64(content)
                            runSsmCommandInline(
                                app_server_instance_id,
                                "\$destFile = 'E:\\apps\\${CLIENT_NAME}\\${relativePath}'; \$bytes = [System.Convert]::FromBase64String('${encoded}'); [System.IO.File]::WriteAllBytes(\$destFile, \$bytes); Write-Output 'Restored: ${relativePath}'",
                                60,
                                false
                            )
                        }
                        echo "All 4 config files restored successfully."

                        // --- Restart both app pools ---
                        echo "Restarting app pools..."
                        runSsmCommandInline(
                            app_server_instance_id,
                            "foreach (\$poolName in @('AppName Web App Pool', '${CLIENT_NAME}')) { if ((Get-WebAppPoolState -Name \$poolName).Value -ne 'Started') { Start-WebAppPool -Name \$poolName }; \$timeoutSec = 30; \$elapsed = 0; while ((Get-WebAppPoolState -Name \$poolName).Value -ne 'Started' -and \$elapsed -lt \$timeoutSec) { Start-Sleep -Seconds 1; \$elapsed++ }; if ((Get-WebAppPoolState -Name \$poolName).Value -ne 'Started') { throw \"App pool '\$poolName' did not start within \$timeoutSec seconds\" } }; Write-Output 'Both app pools started successfully'",
                            60
                        )
                        echo "Both app pools started successfully."
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}