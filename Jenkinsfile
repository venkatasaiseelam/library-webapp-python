pipeline {
    agent {
        label 'docker'
    }
    environment {
        IMAGE_NAME = "devops1117/library"
    }

    stages {
        stage('Code checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/venkatasaiseelam/library-webapp-python.git'
            }
        }
        stage('get SHA as Image Tag') { // This stage retrieves the short SHA of the latest commit and sets it as the IMAGE_TAG environment variable
            steps {
                script {
                    env.IMAGE_TAG = sh (
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                        ).trim()
                }
            }
        }
        stage('Code QA') {
            environment {
                scannerHome = tool 'mysonar'
            }
            steps {
                withSonarQubeEnv('sonarsc') {
                    sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=library-app"
                }
            }
        }
        stage('Image Build') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:auth-v${IMAGE_TAG} auth
                docker build -t ${IMAGE_NAME}:book-v${IMAGE_TAG} book
                docker build -t ${IMAGE_NAME}:borrow-v${IMAGE_TAG} borrow
                docker build -t ${IMAGE_NAME}:db-v${IMAGE_TAG} database
                docker build -t ${IMAGE_NAME}:frontend-v${IMAGE_TAG} .
                """
            }
        }
        stage('Trivy Image Scan') {
            steps {
                sh """
                trivy image ${IMAGE_NAME}:auth-v${IMAGE_TAG} >> auth-report.txt
                trivy image ${IMAGE_NAME}:book-v${IMAGE_TAG} >> book-report.txt
                trivy image ${IMAGE_NAME}:borrow-v${IMAGE_TAG} >> borrow-report.txt
                trivy image ${IMAGE_NAME}:db-v${IMAGE_TAG} >> db-report.txt
                trivy image ${IMAGE_NAME}:frontend-v${IMAGE_TAG} >> frontend-report.txt
                """
            }
        }
        stage("Push to Repository") {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'dockerhub') {
                        sh """
                        docker push ${IMAGE_NAME}:auth-v${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:book-v${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:borrow-v${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:db-v${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:frontend-v${IMAGE_TAG}
                        """
                    }
                }
            }
        }
        stage ('Update k8s Manifest') {
            steps {
                script {
                    withCredentials([gitUsernamePassword(credentialsId: 'github-token', gitToolName: 'Default')]) {
                        dir('kubernetes') {
                            sh """
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:auth-v${IMAGE_TAG}|g' auth-deployment.yaml
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:book-v${IMAGE_TAG}|g' book-deployment.yaml
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:borrow-v${IMAGE_TAG}|g' borrow-deployment.yaml
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:db-v${IMAGE_TAG}|g' db-deployment.yaml
                            sed -i 's|image: .*|image: ${IMAGE_NAME}:frontend-v${IMAGE_TAG}|g' frontend-deployment.yaml
                            """
                            sh """
                            git config user.email "jenkins@devops.com"
                            git config user.name "Jenkins"
                            
                            git add .
                            git commit -m "update images version to latest commit SHA: ${IMAGE_TAG}" || true
                            git push origin main
                            """
                        }
                    }
                }
            }
        }
    }
}
