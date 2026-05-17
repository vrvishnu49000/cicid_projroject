pipeline {
    agent any

    environment {
        DOCKERHUB_USER      = 'YOUR_DOCKERHUB_USERNAME'
        IMAGE_NAME          = 'YOUR_IMAGE_NAME'
        DEV_EC2_HOST        = 'YOUR_DEV_EC2_IP'
        PROD_EC2_HOST       = 'YOUR_PROD_EC2_IP'
        EC2_SSH_USER        = 'ubuntu'
        HOST_PORT           = '80'
        CONTAINER_PORT      = '80'
        DOCKERHUB_CREDS     = 'dockerhub-credentials'   // Jenkins credentials ID
        DEV_SSH_CREDS       = 'dev-ec2-ssh-key'         // Jenkins credentials ID
        PROD_SSH_CREDS      = 'prod-ec2-ssh-key'        // Jenkins credentials ID
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Set Build Info') {
            steps {
                script {
                    env.GIT_SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.BRANCH        = env.GIT_BRANCH?.replaceAll('origin/', '') ?: 'unknown'
                    env.IMAGE_TAG     = "${env.BRANCH}-${env.BUILD_NUMBER}-${env.GIT_SHORT_SHA}"
                    env.FULL_IMAGE    = "${DOCKERHUB_USER}/${IMAGE_NAME}:${env.IMAGE_TAG}"
                    echo "Branch      : ${env.BRANCH}"
                    echo "Image Tag   : ${env.IMAGE_TAG}"
                    echo "Full Image  : ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.build("${env.FULL_IMAGE}")
                }
            }
        }

        // ── DEV FLOW (push to dev branch) ────────────────────────────────────
        stage('Deploy to Dev EC2') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    sshagent(credentials: [DEV_SSH_CREDS]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_SSH_USER}@${DEV_EC2_HOST} '
                                docker pull ${env.FULL_IMAGE} 2>/dev/null || true
                                docker stop nginx-dev 2>/dev/null || true
                                docker rm   nginx-dev 2>/dev/null || true
                                docker run -d \
                                    --name nginx-dev \
                                    --restart always \
                                    -p ${HOST_PORT}:${CONTAINER_PORT} \
                                    ${env.FULL_IMAGE}
                                echo "Deployed ${env.IMAGE_TAG} to DEV"
                            '
                        """
                    }
                }
            }
        }

        // ── PROD FLOW (merge to prod branch) ─────────────────────────────────
        stage('Push Image to Docker Hub') {
            when {
                branch 'prod'
            }
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDS) {
                        docker.image("${env.FULL_IMAGE}").push()
                        // also tag as latest
                        docker.image("${env.FULL_IMAGE}").push('latest')
                    }
                }
            }
        }

        stage('Deploy to Prod EC2') {
            when {
                branch 'prod'
            }
            steps {
                script {
                    sshagent(credentials: [PROD_SSH_CREDS]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_SSH_USER}@${PROD_EC2_HOST} '
                                docker pull ${env.FULL_IMAGE}
                                docker stop nginx-prod 2>/dev/null || true
                                docker rm   nginx-prod 2>/dev/null || true
                                docker run -d \
                                    --name nginx-prod \
                                    --restart always \
                                    -p ${HOST_PORT}:${CONTAINER_PORT} \
                                    ${env.FULL_IMAGE}
                                echo "Deployed ${env.IMAGE_TAG} to PROD"
                            '
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS — Image: ${env.FULL_IMAGE}"
        }
        failure {
            echo "Pipeline FAILED — Branch: ${env.BRANCH}, Build: ${env.BUILD_NUMBER}"
        }
        always {
            // clean up local docker image to save disk space
            sh "docker rmi ${env.FULL_IMAGE} || true"
        }
    }
}
