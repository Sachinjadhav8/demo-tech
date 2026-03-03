pipeline {
    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: ['DEPLOY', 'ROLLBACK'],
            description: 'Select action'
        )
        string(
            name: 'ROLLBACK_TAG',
            defaultValue: 'v1.0',
            description: 'Tag for rollback'
        )
    }

    environment {
        IMAGE_NAME = "YOUR_DOCKERHUB_USERNAME/devops-demo"
        DOCKER_SERVER = "ec2-user@DOCKER_EC2_IP"
    }

    stages {

        stage('Get Tag') {
            when { expression { params.ACTION == 'DEPLOY' } }
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

        stage('Build Image') {
            when { expression { params.ACTION == 'DEPLOY' } }
            steps {
                sh "docker build -t $IMAGE_NAME:$TAG ."
            }
        }

        stage('Docker Login') {
            when { expression { params.ACTION == 'DEPLOY' } }
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
            when { expression { params.ACTION == 'DEPLOY' } }
            steps {
                sh "docker push $IMAGE_NAME:$TAG"
            }
        }

        stage('Deploy') {
            when { expression { params.ACTION == 'DEPLOY' } }
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

        stage('Rollback') {
            when { expression { params.ACTION == 'ROLLBACK' } }
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no $DOCKER_SERVER '
                    docker pull $IMAGE_NAME:${params.ROLLBACK_TAG} &&
                    docker rm -f devops-container || true &&
                    docker run -d -p 80:80 --name devops-container $IMAGE_NAME:${params.ROLLBACK_TAG}
                    '
                    """
                }
            }
        }
    }
}