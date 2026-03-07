pipeline {
    agent any

    environment {
        IMAGE_NAME = "sachin820/devops-demo"
        DOCKER_SERVER = "ec2-user@35.154.2.180"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/Sachinjadhav8/demo-tech.git'
            }
        }

        stage('Get Git Tag') {
            steps {
                script {
                    TAG = sh(
                        script: "git describe --tags --abbrev=0",
                        returnStdout: true
                    ).trim()
                }
                echo "Deploying version: ${TAG}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t $IMAGE_NAME:$TAG ."
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'USER',
                    passwordVariable: 'PASS')]) {

                    sh 'echo $PASS | docker login -u $USER --password-stdin'
                }
            }
        }

        stage('Push Image') {
            steps {
                sh "docker push $IMAGE_NAME:$TAG"
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no $DOCKER_SERVER '
                    docker pull $IMAGE_NAME:$TAG &&
                    docker rm -f devops-container || true &&
                    docker run -d -p 80:80 --name devops-container $IMAGE_NAME:$TAG
                    '
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    userChoice = input(
                        message: "Deployment completed. Verify application. Continue or Rollback?",
                        parameters: [
                            choice(
                                name: 'ACTION',
                                choices: ['CONTINUE', 'ROLLBACK'],
                                description: 'Select next step'
                            )
                        ]
                    )
                }
            }
        }

        stage('Rollback') {
            when {
                expression { userChoice == 'ROLLBACK' }
            }
            steps {
                script {
                    rollbackTag = input(
                        message: "Enter tag to rollback",
                        parameters: [
                            string(
                                name: 'ROLLBACK_TAG',
                                defaultValue: 'v1.0',
                                description: 'Docker image tag'
                            )
                        ]
                    )
                }

                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no $DOCKER_SERVER '
                    docker pull $IMAGE_NAME:${rollbackTag} &&
                    docker rm -f devops-container || true &&
                    docker run -d -p 80:80 --name devops-container $IMAGE_NAME:${rollbackTag}
                    '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully 🚀"
        }
        failure {
            echo "Pipeline failed ❌"
        }
    }
}