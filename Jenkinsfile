pipeline {
    agent any

    environment {
        AWS_REGION                  = 'ap-southeast-1' 
        AWS_ACCOUNT_ID              = '001746151322' 
        FRONTEND_ECR_REPOSITORY_NAME= 'bndz/frontend' 
        BACKEND_ECR_REPOSITORY_NAME = 'bndz/backend'  
        EKS_CLUSTER_NAME            = 'my-eks-cluster'  
        K8S_NAMESPACE               = 'default' 
        FRONTEND_ECR_URL            = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${FRONTEND_ECR_REPOSITORY_NAME}"
        BACKEND_ECR_URL             = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${BACKEND_ECR_REPOSITORY_NAME}"
        KUBECONFIG_PATH             = '/var/lib/jenkins/.kube/config'
        MONGODB_IMAGE               = "mongo:6.0"
    }

    stages {
        stage('Checkout') {
            steps {
                retry(2) {
                    checkout scm
                }
                script {
                    env.BUILD_TAG = "ver-${BUILD_NUMBER}"
                    echo "Build Tag: ${env.BUILD_TAG}"
                }
            }
        }

        stage('Setup AWS CLI and Kubectl Context') {
            steps {
                sh "aws --version"
                sh "kubectl version --client"
                sh "kubectl --kubeconfig ${env.KUBECONFIG_PATH} config current-context"
                sh "kubectl --kubeconfig ${env.KUBECONFIG_PATH} get nodes -o wide"
            }
        }

        stage('Build and Push Docker Images') {
            parallel {
                stage('Build & Push Backend Image') {
                    when { expression { fileExists 'backend/Dockerfile' } }
                    steps {
                        dir('backend') {
                            script {
                                def fullImageName = "${env.BACKEND_ECR_URL}:${env.BUILD_TAG}"
                                echo "Building Backend Image: ${fullImageName}"
                                sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                                docker.build(fullImageName, ".")
                                echo "Pushing Backend Image: ${fullImageName}"
                                docker.image(fullImageName).push()
                            }
                        }
                    }
                }

                stage('Build & Push Frontend Image') {
                    when { expression { fileExists 'frontend/Dockerfile' } }
                    steps {
                        dir('frontend') {
                            script {
                                def fullImageName = "${env.FRONTEND_ECR_URL}:${env.BUILD_TAG}"
                                echo "Building Frontend Image: ${fullImageName}"
                                sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com"
                                docker.build(fullImageName, ".")
                                echo "Pushing Frontend Image: ${fullImageName}"
                                docker.image(fullImageName).push()
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to EKS') {
            steps {
                script {
                    // Deploy or update MongoDB
                    sh """
                        if kubectl --kubeconfig ${env.KUBECONFIG_PATH} get statefulset mongo -n ${env.K8S_NAMESPACE} >/dev/null 2>&1; then
                            echo "Updating existing MongoDB StatefulSet image..."
                            kubectl --kubeconfig ${env.KUBECONFIG_PATH} set image statefulset/mongo mongo=${env.MONGODB_IMAGE} -n ${env.K8S_NAMESPACE}
                        else
                            echo "MongoDB StatefulSet not found — creating new one..."
                            kubectl --kubeconfig ${env.KUBECONFIG_PATH} apply -f Manifest-AWS/mongodb.yaml -n ${env.K8S_NAMESPACE}
                        fi
                    """

                    // Deploy Backend
                    env.BACKEND_IMAGE_URI = "${env.BACKEND_ECR_URL}:${env.BUILD_TAG}"
                    if (env.BACKEND_IMAGE_URI) {
                        echo "Deploying Backend: ${env.BACKEND_IMAGE_URI}"
                        sh """
                            sed -i 's|image:.*${env.BACKEND_ECR_REPOSITORY_NAME}:.*|image: ${env.BACKEND_IMAGE_URI}|g' Manifest-AWS/backend.yaml
                            kubectl --kubeconfig ${env.KUBECONFIG_PATH} apply -f Manifest-AWS/backend.yaml -n ${env.K8S_NAMESPACE}
                        """
                    } else {
                        echo "Backend image URI not found. Skipping backend deployment."
                    }

                    // Deploy Frontend
                    env.FRONTEND_IMAGE_URI = "${env.FRONTEND_ECR_URL}:${env.BUILD_TAG}"
                    if (env.FRONTEND_IMAGE_URI) {
                        echo "Deploying Frontend: ${env.FRONTEND_IMAGE_URI}"
                        sh """
                            sed -i 's|image:.*${env.FRONTEND_ECR_REPOSITORY_NAME}:.*|image: ${env.FRONTEND_IMAGE_URI}|g' Manifest-AWS/frontend.yaml
                            kubectl --kubeconfig ${env.KUBECONFIG_PATH} apply -f Manifest-AWS/frontend.yaml -n ${env.K8S_NAMESPACE}
                        """
                    } else {
                        echo "Frontend image URI not found. Skipping frontend deployment."
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
            cleanWs()
        }
        success {
            echo 'Pipeline Succeeded!'
        }
        failure {
            echo 'Pipeline Failed!'
        }
    }
}
