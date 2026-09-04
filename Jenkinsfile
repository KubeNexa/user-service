pipeline{
    agent any

    environment {
        IMAGE_NAME = "manoj_s/microservices-platform/user-service:${GIT_COMMIT}"
    }

    stages{
        stage('Git Checkout'){
            steps{
                git url: 'https://github.com/KubeNexa/user-service.git' , branch: 'main'
            }
        }

        stage('Run Unit Tests'){
            steps {
                sh 'mvn test'
            }
        }

        stage('Build Docker Image'){
            steps{
                sh '''
                    printenv
                    docker build -t ${IMAGE_NAME} .
                '''
            }
        }

        stage('Login to Docker Hub'){
            steps{
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub-credentials', 
                    usernameVariable: 'DOCKER_USERNAME', 
                    passwordVariable: 'DOCKER_PASSWORD'
                    )]) 
                    {
                    sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                }
            }
        }

        stage('Push Docker Image to DockerHub'){
            steps{
                sh 'docker push ${IMAGE_NAME}'
            }
        }

        stage('Update GitOps Deployment'){
            steps{
                withCredentials([usernamePassword(
                    credentialsId: 'gitops-repo-credentials', 
                    usernameVariable: 'GITOPS_USERNAME', 
                    passwordVariable: 'GITOPS_PASSWORD'
                    )]) 
                    {
                    sh '''
                        if [ -d "gitops" ]; then
                            echo "Directory exists, removing it..."
                            rm -rf gitops
                        fi
                        git clone https://$GITOPS_USERNAME:$GIT_PASSWORD@github.com/KubeNexa/GitOps.git gitops
                        cd gitops/base/apigatewayservice

                        git config user.email "jenkins@ci.com"
                        git config user.name "jenkins"

                        # Update the image tag in the kustomization.yaml file
                        sed -i "s|image: .*apigateway.*|image: ${IMAGE_NAME}|g" deployment.yaml
                       
                       git add .
                       git commit -m "Update API Gateway image to ${IMAGE_NAME}"
                       git push origin main
                    '''
                    }
            }
        }

        post {
            always {
                sh 'docker rmi ${IMAGE_NAME} || true'
                sh 'docker logout || true'
            }
            
            success {
                echo "Build and push successful: ${IMAGE_NAME}"
            }

            failure {
                echo "Pipeline failed. Check the logs above."
            }
        }

    }