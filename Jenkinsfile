pipeline {
    agent {
        label 'worker'
    }

    environment {
        IMAGE_NAME = "kavyakeerthid/my-frontend-app"
        IMAGE_TAG = "${BUILD_NUMBER}"

        K8S_HOST = "172.31.11.157"
        HELM_RELEASE = "my-frontend"
        HELM_CHART = "/home/ubuntu/my-frontend-chart"
        SSH_KEY = "/home/ubuntu/.ssh/k8s_deploy_key"
    }

    stages {

        stage('Clone Repository') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/lskavyakeerthi/august.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Building Docker image"
                    echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "=============================================="

                    docker build \
                        -t ${IMAGE_NAME}:${IMAGE_TAG} .

                    docker tag \
                        ${IMAGE_NAME}:${IMAGE_TAG} \
                        ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Docker Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login \
                            -u "$DOCKER_USERNAME" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Pushing Docker image"
                    echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "=============================================="

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}

                    docker push ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Deploying to Kubernetes"
                    echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    echo "=============================================="

                    ssh -i ${SSH_KEY} \
                        -o StrictHostKeyChecking=no \
                        ubuntu@${K8S_HOST} \
                        "helm upgrade --install ${HELM_RELEASE} ${HELM_CHART} \
                        --set image.repository=${IMAGE_NAME} \
                        --set image.tag=${IMAGE_TAG}"
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Checking Kubernetes rollout"
                    echo "=============================================="

                    ssh -i ${SSH_KEY} \
                        -o StrictHostKeyChecking=no \
                        ubuntu@${K8S_HOST} \
                        "kubectl rollout status \
                        deployment/my-frontend-mydeploy \
                        --timeout=180s"
                '''
            }
        }

        stage('Verify Image') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Checking deployed image"
                    echo "=============================================="

                    ssh -i ${SSH_KEY} \
                        -o StrictHostKeyChecking=no \
                        ubuntu@${K8S_HOST} \
                        "kubectl get deployment my-frontend-mydeploy \
                        -o=jsonpath='{.spec.template.spec.containers[0].image}'"

                    echo ""
                '''
            }
        }
    }

    post {
        success {
            echo '=============================================='
            echo 'SUCCESS!'
            echo 'Docker image pushed successfully!'
            echo 'Kubernetes deployment completed successfully!'
            echo '=============================================='
        }

        failure {
            echo '=============================================='
            echo 'PIPELINE FAILED!'
            echo 'Check the failed stage in Console Output.'
            echo '=============================================='
        }
    }
}
