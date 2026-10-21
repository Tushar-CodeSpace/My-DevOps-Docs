
pipeline {
    agent any

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
                    url: 'https://github.com/<ORG>/<REPO>.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        "$SCANNER_HOME/bin/sonar-scanner" \
                          -Dsonar.projectName=<Project Name> \
                          -Dsonar.projectKey=<Project Key>
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
