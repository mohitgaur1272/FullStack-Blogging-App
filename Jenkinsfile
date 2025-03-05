pipeline {
    agent any
    
    tools {
        jdk "java17"
        maven "maven3"
    }
    
    environment {
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
        SONAR_TOKEN = credentials('sonarqube_new_token')
        AWS_DEFAULT_REGION = 'ap-south-1'
        ECR_REPO_URL = '774305587583.dkr.ecr.ap-south-1.amazonaws.com'
        SCANNER_HOME = tool "sonar-scanner"
        DOCKER_CREDENTIALS_ID = "docker-cred"
        NAMESPACE = 'default'
        EKS_CLUSTER_NAME = 'new-w3cluster'
        CONTAINER_NAME = "java-container"

        // Dynamic values based on branch
        IMAGE_TAG = ""
        DEPLOYMENT_FILE = ""
    }
    
    stages {
        stage('Set Environment Variables') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        IMAGE_TAG = "java-image:main${BUILD_NUMBER}"
                        DEPLOYMENT_FILE = "deployment-main.yaml"
                    } else if (env.BRANCH_NAME == 'dev') {
                        IMAGE_TAG = "java-image:dev${BUILD_NUMBER}"
                        DEPLOYMENT_FILE = "deployment-dev.yaml"
                    } else if (env.BRANCH_NAME == 'test') {
                        IMAGE_TAG = "java-image:test${BUILD_NUMBER}"
                        DEPLOYMENT_FILE = "deployment-test.yaml"
                    } else {
                        error("Unknown branch, stopping pipeline!")
                    }
                    
                    echo "Branch: ${env.BRANCH_NAME}"
                    echo "Image Tag: ${IMAGE_TAG}"
                    echo "Deployment File: ${DEPLOYMENT_FILE}"
                }
            }
        }
        
        stage('Git Checkout') {
            steps {
                git branch: env.BRANCH_NAME, credentialsId: 'git-cred', url: 'https://github.com/mohitgaur1272/FullStack-Blogging-App.git'
            }
        }
        
        stage('Compile and Test') {
            steps {
                sh 'mvn compile'
                sh 'mvn test'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''trivy fs --format table -o fs-main-$(date +%Y-%m-%d).txt .'''
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                        sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=mohit_ka_project -Dsonar.projectKey=mohit_ki_key -Dsonar.sources=src -Dsonar.java.binaries=target '''
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn package'
            }
        }
        
        stage('Docker Build and Tag') {
            steps {
                script {
                    def dockerImage = "${ECR_REPO_URL}/${IMAGE_TAG}"
                    sh "docker build -t ${dockerImage} ."
                    sh "docker tag ${dockerImage} ${ECR_REPO_URL}/${IMAGE_TAG}"
                }
            }
        }

        stage('Image Scan') {
            steps {
                script {
                  def currentDate = sh(script: "date +%Y-%m-%d", returnStdout: true).trim()
                  sh "trivy image --format table -o main-image-report-${currentDate}.txt ${ECR_REPO_URL}/${IMAGE_TAG}"
                  //sh '''trivy image --format table -o main-image-report-$(date +%Y-%m-%d).txt ${ECR_REPO_URL}/${IMAGE_TAG}'''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URL}"
                sh "docker push ${ECR_REPO_URL}/${IMAGE_TAG}"
                sh "docker rmi -f \$(docker images -q ${ECR_REPO_URL}/${IMAGE_TAG}) || true"
                sh "docker system prune -af"
            }
        }

        stage('Add Kubeconfig to Jenkins') {
            steps {
                sh "aws eks --region ${AWS_DEFAULT_REGION} update-kubeconfig --name ${EKS_CLUSTER_NAME}"
            }
        }

        stage('Update Deployment File') {
            steps {
                sh "sed -i 's|image: 774305587583.dkr.ecr.ap-south-1.amazonaws.com/java-image:latest|image: ${ECR_REPO_URL}/${IMAGE_TAG}|' ${DEPLOYMENT_FILE}"
            }
        }

        stage('Apply Deployment') {
            steps {
                sh "kubectl apply -f ${DEPLOYMENT_FILE} --namespace=${NAMESPACE}"
            }
        }
    }
}
