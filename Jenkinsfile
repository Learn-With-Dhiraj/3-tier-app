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
                    credentialsId: 'github-token',
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
                sh 'docker build -t ${APP_NAME}:${BUILD_NUMBER} .'
            }
        }
        
        stage('Trivy Scan') {
            steps {
                sh '''
                    trivy image \
                    --format table \
                    -o trivy-report.html \
                    ${APP_NAME}:${BUILD_NUMBER}
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
                        
                        docker tag ${APP_NAME}:${BUILD_NUMBER} \
                        ${ECR_URL}/${APP_NAME}:${BUILD_NUMBER}
                        
                        docker push ${ECR_URL}/${APP_NAME}:${BUILD_NUMBER}
                    '''
                }
            }
        }
        
        stage('Update Manifest') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-token',
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {
                    sh '''
                        rm -rf 3-tier-manifests
                        git clone https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/Learn-With-Dhiraj/3-tier-manifests.git
                        cd 3-tier-manifests
                        sed -i "s/tag:.*/tag: ${BUILD_NUMBER}/" dev/values.yaml
                        git config user.email "jenkins@devops.com"
                        git config user.name "Jenkins"
                        git add .
                        git commit -m "Update dev image tag to ${BUILD_NUMBER}"
                        git push
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
