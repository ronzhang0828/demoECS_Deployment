pipeline {
    agent any
    environment {
        AWS_REGION = 'ap-southeast-2'
        ECR_REPO_FRONTEND = 'demoreact'
        ECR_REPO_BACKEND = 'demoecs'
        IMAGE_TAG = "${BUILD_NUMBER}"
        GIT_REPO = 'git@github.com:ronzhang0828/demoECS_Deployment.git'
        GIT_BRANCH = 'deploy'
        ECR_REGISTRY = "541730394177.dkr.ecr.ap-southeast-2.amazonaws.com"
        ECS_CLUSTER = 'your-ecs-cluster'
        ECS_SERVICE_FRONTEND = 'your-frontend-service'
        ECS_SERVICE_BACKEND = 'your-backend-service'
    }

    stages {
        stage('Checkout Code') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${GIT_BRANCH}"]],
                        userRemoteConfigs: [[
                            credentialsId: 'GITHUB_SSH',
                            url: GIT_REPO
                        ]]
                    ])
                }
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changedFiles = sh(script: "git diff --name-only HEAD~1", returnStdout: true).trim()
                    echo "Changed files: ${changedFiles}"
                    
                    env.BUILD_FRONTEND = changedFiles.contains("frontend/") ? "true" : "false"
                    env.BUILD_BACKEND = changedFiles.contains("backend/") ? "true" : "false"

                    echo "Build Frontend: ${env.BUILD_FRONTEND}"
                    echo "Build Backend: ${env.BUILD_BACKEND}"
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withAWS(credentials: 'JenkinsAccessCredential', region: "${AWS_REGION}") {
                    sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    """
                }
            }
        }

        stage('Build and Push Frontend Docker Image') {
            when { expression { env.BUILD_FRONTEND == "true" } }
            steps {
                script {
                    echo "Building Frontend..."
                    sh """
                    cd frontend
                    docker build -t ${ECR_REPO_FRONTEND}:${IMAGE_TAG} .
                    docker tag ${ECR_REPO_FRONTEND}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO_FRONTEND}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ECR_REPO_FRONTEND}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Build and Push Backend Docker Image') {
            when { expression { env.BUILD_BACKEND == "true" } }
            steps {
                script {
                    echo "Building Backend..."
                    sh """
                    cd backend
                    docker build -t ${ECR_REPO_BACKEND}:${IMAGE_TAG} .
                    docker tag ${ECR_REPO_BACKEND}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO_BACKEND}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${ECR_REPO_BACKEND}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy Frontend to ECS') {
            when { expression { env.BUILD_FRONTEND == "true" } }
            steps {
                withAWS(credentials: 'JenkinsAccessCredential', region: "${AWS_REGION}") {
                    sh """
                    aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE_FRONTEND} --force-new-deployment
                    """
                }
            }
        }

        stage('Deploy Backend to ECS') {
            when { expression { env.BUILD_BACKEND == "true" } }
            steps {
                withAWS(credentials: 'JenkinsAccessCredential', region: "${AWS_REGION}") {
                    sh """
                    aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE_BACKEND} --force-new-deployment
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up Docker images and containers..."
            sh """
            docker system prune -af --volumes
            """
            echo "Cleaning up Jenkins workspace..."
            deleteDir()
        }
    }
}
