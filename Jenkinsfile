
pipeline {
    agent any

    options {
        disableConcurrentBuilds()
    }

    tools {
        nodejs 'nodejs24'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        QUALITY_GATE_STATUS = 'NOT_REACHED'
    }

    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'master',
                    credentialsId: 'github-pat',
                    url: 'https://github.com/NIDO-MACHINERIES/nidoworkz-backend.git'

                script {
                    env.GIT_COMMIT = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    env.GIT_BRANCH = sh(
                        script: 'git branch --show-current',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        "$SCANNER_HOME/bin/sonar-scanner" \
                          -Dsonar.projectName=nidoworkz-backend \
                          -Dsonar.projectKey=nidoworkz-backend
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        env.QUALITY_GATE_STATUS = qg.status

                        echo "SonarQube Quality Gate: ${env.QUALITY_GATE_STATUS}"

                        if (qg.status != 'OK') {
                            error "Quality Gate failed: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('TRIVY FS SCAN') {
            steps {
                sh '''
                    trivy fs \
                        --scanners vuln,misconfig,secret \
                        --format table \
                        --output trivyfs.txt \
                        --no-progress \
                        .
                '''
            }
        }

        stage('Run Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Run Tests') {
            steps {
                sh 'npm run test'
            }
        }
    }


    post {
        always {
            script {
                def buildStatus = currentBuild.currentResult ?: 'UNKNOWN'
                def gateStatus = env.QUALITY_GATE_STATUS ?: 'NOT_REACHED'
                def commit = env.GIT_COMMIT ?: 'Not available'
                def branch = env.GIT_BRANCH ?: 'Not available'
                def duration = currentBuild.durationString ?: 'Not available'

                def testSummary = null
                def testStatus = 'Not available'
                def testRows = ''

                // Publish compatible JUnit reports when available.
                try {
                    testSummary = junit(
                        testResults: '**/junit*.xml,**/test-results/**/*.xml,**/test-results/*.xml',
                        allowEmptyResults: true
                    )

                    if (testSummary != null && testSummary.totalCount > 0) {
                        def passed = testSummary.totalCount -
                            testSummary.failCount -
                            testSummary.skipCount

                        testStatus = 'Available'

                        testRows = """
                            <tr><td>Total</td><td>${testSummary.totalCount}</td></tr>
                            <tr><td>Passed</td><td>${passed}</td></tr>
                            <tr><td>Failed</td><td>${testSummary.failCount}</td></tr>
                            <tr><td>Skipped</td><td>${testSummary.skipCount}</td></tr>
                        """
                    } else {
                        testStatus = 'No test cases found in compatible JUnit XML reports'
                    }
                } catch (e) {
                    testStatus = 'Report publishing failed'
                    echo "WARNING: JUnit publishing failed: ${e.getMessage()}"
                }

                def trivyStatus = fileExists('trivyfs.txt') ?
                    'Generated' : 'Not available'

                def dependencyStatus = fileExists('dependency-check-report.html') ||
                    fileExists('dependency-check-report.xml') ||
                    fileExists('dependency-check-report.json') ?
                    'Generated' : 'Not generated / not configured'

                // Archive reports independently of email delivery.
                try {
                    archiveArtifacts(
                        artifacts: 'trivyfs.txt,**/dependency-check-report.*,**/junit*.xml,**/test-results/**/*.xml',
                        allowEmptyArchive: true,
                        fingerprint: true
                    )
                } catch (e) {
                    echo "WARNING: Artifact archiving failed: ${e.getMessage()}"
                }

                def statusColor = buildStatus == 'SUCCESS' ?
                    '#15803d' :
                    (buildStatus == 'UNSTABLE' ? '#b45309' : '#b91c1c')

                def gateColor = gateStatus == 'OK' ?
                    '#15803d' :
                    (gateStatus == 'NOT_REACHED' ? '#b45309' : '#b91c1c')

                def gateMessage = gateStatus == 'OK' ?
                    'Quality Gate passed' :
                    (gateStatus == 'NOT_REACHED' ?
                        'No Quality Gate result recorded; do not treat as passed' :
                        "Quality Gate: ${gateStatus}")

                def emailBody = """
                <html>
                <body style="margin:0;padding:20px;background:#f1f5f9;
                    font-family:Arial,Helvetica,sans-serif;color:#1e293b;">
                <div style="max-width:700px;margin:auto;background:#ffffff;
                    border:1px solid #e2e8f0;border-radius:10px;overflow:hidden;">

                    <div style="background:#172554;color:white;padding:24px 28px;">
                    <div style="font-size:12px;letter-spacing:1px;">
                        NIDO MACHINERIES
                    </div>
                    <h1 style="font-size:24px;margin:10px 0;">
                        NIDOWORKZ Backend
                    </h1>
                    <div>CI/CD Summary Report</div>
                    <div style="margin-top:16px;display:inline-block;
                        background:${statusColor};padding:8px 12px;
                        border-radius:5px;font-weight:bold;">
                        ${buildStatus}
                    </div>
                    </div>

                    <div style="padding:24px 28px;">
                    <p style="line-height:1.6;">
                        The automated CI/CD pipeline has completed. Below is the
                        summary of build, code quality, security scanning and
                        available test results. Detailed reports are attached
                        to this email where available.
                    </p>

                    <h2 style="font-size:18px;">Build Summary</h2>
                    <table width="100%" cellpadding="10" cellspacing="0"
                        style="border-collapse:collapse;font-size:14px;">
                        <tr style="background:#f8fafc;">
                        <td style="border:1px solid #e2e8f0;"><b>Project</b></td>
                        <td style="border:1px solid #e2e8f0;">nidoworkz-backend</td>
                        </tr>
                        <tr>
                        <td style="border:1px solid #e2e8f0;"><b>Build</b></td>
                        <td style="border:1px solid #e2e8f0;">#${env.BUILD_NUMBER}</td>
                        </tr>
                        <tr style="background:#f8fafc;">
                        <td style="border:1px solid #e2e8f0;"><b>Branch</b></td>
                        <td style="border:1px solid #e2e8f0;">${branch}</td>
                        </tr>
                        <tr>
                        <td style="border:1px solid #e2e8f0;"><b>Commit</b></td>
                        <td style="border:1px solid #e2e8f0;">${commit}</td>
                        </tr>
                        <tr style="background:#f8fafc;">
                        <td style="border:1px solid #e2e8f0;"><b>Duration</b></td>
                        <td style="border:1px solid #e2e8f0;">${duration}</td>
                        </tr>
                    </table>

                    <h2 style="font-size:18px;margin-top:24px;">Quality &amp; Security</h2>
                    <table width="100%" cellpadding="10" cellspacing="0"
                        style="border-collapse:collapse;font-size:14px;">
                        <tr>
                        <td style="border:1px solid #e2e8f0;">
                            <b>SonarQube Quality Gate</b>
                        </td>
                        <td style="border:1px solid #e2e8f0;
                            color:${gateColor};font-weight:bold;">
                            ${gateStatus}
                        </td>
                        </tr>
                        <tr>
                        <td style="border:1px solid #e2e8f0;">
                            <b>Quality Gate interpretation</b>
                        </td>
                        <td style="border:1px solid #e2e8f0;">
                            ${gateMessage}
                        </td>
                        </tr>
                        <tr>
                        <td style="border:1px solid #e2e8f0;">
                            <b>Trivy filesystem scan</b>
                        </td>
                        <td style="border:1px solid #e2e8f0;">
                            ${trivyStatus}
                        </td>
                        </tr>
                        <tr>
                        <td style="border:1px solid #e2e8f0;">
                            <b>Dependency Check</b>
                        </td>
                        <td style="border:1px solid #e2e8f0;">
                            ${dependencyStatus}
                        </td>
                        </tr>
                    </table>

                    <h2 style="font-size:18px;margin-top:24px;">Test Summary</h2>
                    <p>${testStatus}</p>
                    <table width="100%" cellpadding="9" cellspacing="0"
                        style="border-collapse:collapse;font-size:14px;">
                        ${testRows ?: '<tr><td style="border:1px solid #e2e8f0;">Detailed test counts are unavailable because no compatible JUnit XML results were found.</td></tr>'}
                    </table>

                    <h2 style="font-size:18px;margin-top:24px;">Email Attachments</h2>
                    <ul style="line-height:1.8;">
                        <li>Trivy filesystem scan report, if generated</li>
                        <li>JUnit XML reports, if generated</li>
                        <li>OWASP Dependency Check reports, if generated</li>
                        <li>Compressed Jenkins build log</li>
                    </ul>

                    <p style="font-size:13px;color:#475569;line-height:1.6;">
                        A generated security report does not necessarily mean
                        that no findings exist. Review the attached reports
                        before approving a release.
                    </p>
                    </div>

                    <div style="padding:16px 28px;background:#f8fafc;
                        border-top:1px solid #e2e8f0;font-size:12px;color:#64748b;">
                    Automatically generated by Jenkins CI/CD.
                    <br/>NIDOWORKZ Backend — Engineering
                    </div>
                </div>
                </body>
                </html>
                """

                try {
                    emailext(
                        to: 'demo1@email.com, demo2@email.com',
                        subject: "[${buildStatus}] NIDOWORKZ Backend - Build #${env.BUILD_NUMBER} - CI/CD Summary",
                        mimeType: 'text/html',
                        body: emailBody,
                        attachmentsPattern: 'trivyfs.txt,**/dependency-check-report.*,**/junit*.xml,**/test-results/**/*.xml',
                        attachLog: true,
                        compressLog: true
                    )

                    echo 'Stakeholder summary email submitted.'
                } catch (e) {
                    echo "WARNING: Email notification failed: ${e.getMessage()}"
                }
            }
        }
    }
}
