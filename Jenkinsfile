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

        // stage('OWASP FS SCAN') {
        //     steps {
        //         dependencyCheck(
        //             odcInstallation: 'DP-Check',
        //             nvdCredentialsId: 'nvd-api-key',
        //             additionalArguments: '--scan . --disableYarnAudit --disableNodeAudit --format XML --out . --nvdValidForHours 24'
        //         )
        //     }
        //     post {
        //         always {
        //             dependencyCheckPublisher(
        //                 pattern: '**/dependency-check-report.xml',
        //                 skipNoReportFiles: true
        //             )
        //         }
        //     }
        // }

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
}