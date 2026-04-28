pipeline {
    agent any
    environment {
        ECR_URL = "040235853848.dkr.ecr.ap-south-1.amazonaws.com"
        APP_NAME = "devops-app"
        AWS_REGION = "ap-south-1"
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'develop',
                    credentialsId: 'github-manifests-token',
                    url: 'https://github.com/Learn-With-Dhiraj/3-tier-app.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=three-tier-app \
                        -Dsonar.projectName=three-tier-app \
                        -Dsonar.sources=.
                    '''
                }
            }
        }
        stage('OWASP Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./',
                                odcInstallation: 'owasp-check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Docker Build') {
            steps {
                sh 'docker build -t ${APP_NAME}:latest .'
            }
        }
        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image \
                    --format table \
                    -o trivy-report.html \
                    ${APP_NAME}:latest
                '''
            }
        }
        stage('ECR Push') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS \
                        --password-stdin ${ECR_URL}
                        
                        docker tag ${APP_NAME}:latest \
                        ${ECR_URL}/${APP_NAME}:latest
                        
                        docker push ${ECR_URL}/${APP_NAME}:latest
                    '''
                }
            }
        }
        stage('Restart Dev Deployment') {
            steps {
                withAWS(credentials: 'aws-credentials', region: "${AWS_REGION}") {
                    sh '''
                        aws eks update-kubeconfig \
                        --name devops \
                        --region ${AWS_REGION}
                        
                        kubectl rollout restart deployment \
                        three-tier-dev-app -n dev
                        
                        kubectl rollout status deployment \
                        three-tier-dev-app -n dev
                    '''
                }
            }
        }
    }
    post {
        success {
            echo 'Pipeline Successfully Completed! ✅'
        }
        failure {
            echo 'Pipeline Failed! ❌'
        }
    }
}
