pipeline {
    agent any
    environment {
        ECR_URL = "://amazonaws.com"
        APP_NAME = "devops-app"
        AWS_REGION = "ap-south-1"
        SCANNER_HOME = tool 'sonar-scanner'
        // GitHub Credentials ID जो Jenkins मध्ये सेव्ह केला आहे
        GIT_CRED_ID = 'github-manifests-token' 
    }
    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'develop',
                    credentialsId: "${GIT_CRED_ID}",
                    url: 'https://github.com'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube-server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=three-tier-app -Dsonar.sources=."
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    // ECR ला लॉगिन करा
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URL}"
                    
                    // इमेज बिल्ड करा (Build Number चा टॅग देऊन)
                    sh "docker build -t ${APP_NAME}:${BUILD_NUMBER} ."
                    
                    // ECR साठी टॅग करा आणि पुश करा
                    sh "docker tag ${APP_NAME}:${BUILD_NUMBER} ${ECR_URL}/${APP_NAME}:${BUILD_NUMBER}"
                    sh "docker tag ${APP_NAME}:${BUILD_NUMBER} ${ECR_URL}/${APP_NAME}:latest"
                    sh "docker push ${ECR_URL}/${APP_NAME}:${BUILD_NUMBER}"
                    sh "docker push ${ECR_URL}/${APP_NAME}:latest"
                }
            }
        }

        stage('Update Manifest in GitHub') {
            steps {
                script {
                    // Git कॉन्फिगरेशन
                    sh "git config user.email 'jenkins@devops.com'"
                    sh "git config user.name 'Jenkins-CI'"

                    // values.yaml मध्ये टॅग अपडेट करा (sed कमांड वापरून)
                    sh "sed -i 's/tag: .*/tag: ${BUILD_NUMBER}/' values.yaml"

                    // बदल पुश करा
                    withCredentials([usernamePassword(credentialsId: "${GIT_CRED_ID}", passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        sh """
                            git add values.yaml
                            git commit -m "chore: update image tag to ${BUILD_NUMBER} [skip ci]"
                            git push https://${GIT_PASSWORD}@://github.com develop
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            // क्लीनअप: लोकल इमेजेस डिलीट करा जेणेकरून डिस्क स्पेस भरून जाणार नाही
            sh "docker rmi -f ${APP_NAME}:${BUILD_NUMBER} ${ECR_URL}/${APP_NAME}:${BUILD_NUMBER} || true"
        }
    }
}

