pipeline {
    agent any
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        IMAGE_NAME = 'imassenda/web-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        // CI (Windows)
        stage('Build') {
            steps {
                echo 'Clonage du projet et construction de l\'image Docker'
                bat 'docker build -t %IMAGE_NAME%:%IMAGE_TAG% .'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Test de l\'image Docker'
                bat '''
                    docker run -d -p 8085:80 --name test-container %IMAGE_NAME%:%IMAGE_TAG%
                    timeout /t 5
                    powershell -Command "if ((Invoke-WebRequest -Uri http://localhost:8085).Content -notmatch 'Welcome') { exit 1 }"
                    docker stop test-container
                    docker rm test-container
                '''
            }
        }
        
        stage('Release') {
            steps {
                echo 'Publication de l\'image sur Docker Hub'
                bat '''
                    echo %DOCKER_HUB_CREDENTIALS_PSW% | docker login -u %DOCKER_HUB_CREDENTIALS_USR% --password-stdin
                    docker push %IMAGE_NAME%:%IMAGE_TAG%
                    docker tag %IMAGE_NAME%:%IMAGE_TAG% %IMAGE_NAME%:latest
                    docker push %IMAGE_NAME%:latest
                '''
            }
        }
        
        // CD (Linux)
        stage('Deploy in Review') {
            steps {
                echo 'Déploiement en environnement de revue'
                withCredentials([sshUserPrivateKey(credentialsId: 'aws-review-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        chmod 600 "$SSH_KEY"
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@16.171.1.77 "
                            sudo docker pull $IMAGE_NAME:$IMAGE_TAG &&
                            sudo docker stop review-container || true &&
                            sudo docker rm review-container || true &&
                            sudo docker run -d -p 80:80 --name review-container $IMAGE_NAME:$IMAGE_TAG
                        "
                    '''
                }
            }
        }
        
        stage('Deploy in Staging') {
            steps {
                echo 'Déploiement en environnement de préproduction'
                withCredentials([sshUserPrivateKey(credentialsId: 'aws-staging-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@51.21.181.0 "
                            sudo docker pull $IMAGE_NAME:$IMAGE_TAG &&
                            sudo docker stop staging-container || true &&
                            sudo docker rm staging-container || true &&
                            sudo docker run -d -p 80:80 --name staging-container $IMAGE_NAME:$IMAGE_TAG
                        "
                    '''
                }
            }
        }
        
        stage('Deploy in Production') {
            steps {
                echo 'Déploiement en environnement de production'
                withCredentials([sshUserPrivateKey(credentialsId: 'aws-production-ssh-key', keyFileVariable: 'SSH_KEY')]) {
                    sh '''
                        ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no ubuntu@13.60.12.219 "
                            sudo docker pull $IMAGE_NAME:$IMAGE_TAG &&
                            sudo docker stop production-container || true &&
                            sudo docker rm production-container || true &&
                            sudo docker run -d -p 80:80 --name production-container $IMAGE_NAME:$IMAGE_TAG
                        "
                    '''
                }
            }
        }
    }
}
