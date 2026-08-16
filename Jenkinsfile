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
                echo '=============================================='
                echo 'Cloning GitHub Repository'
                echo '=============================================='

                git branch: 'master',
                    url: 'https://github.com/lskavyakeerthi/august.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Building Docker Image"
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

        stage('BVT Test') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Running BVT Test"
                    echo "=============================================="

                    echo "Starting test container..."

                    docker run -d \
                        --name bvt-${BUILD_NUMBER} \
                        -p 8080:80 \
                        ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "Waiting for application to start..."
                    sleep 5

                    echo "Checking application availability..."

                    curl -f http://localhost:8080/

                    echo "=============================================="
                    echo "BVT PASSED"
                    echo "Application is up and responding"
                    echo "=============================================="
                '''
            }

            post {
                always {
                    sh '''
                        echo "Cleaning BVT container..."

                        docker rm -f bvt-${BUILD_NUMBER} 2>/dev/null || true
                    '''
                }
            }
        }

        stage('Docker Login') {
            steps {
                echo '=============================================='
                echo 'Docker Hub Login'
                echo '=============================================='

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
                    echo "Pushing Docker Image"
                    echo "=============================================="

                    echo "Pushing ${IMAGE_NAME}:${IMAGE_TAG}"

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}

                    echo "Pushing ${IMAGE_NAME}:latest"

                    docker push ${IMAGE_NAME}:latest

                    echo "Docker image pushed successfully!"
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Deploying to Kubernetes using Helm"
                    echo "=============================================="

                    echo "Image: ${IMAGE_NAME}:${IMAGE_TAG}"

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
                    echo "Checking Kubernetes Rollout"
                    echo "=============================================="

                    ssh -i ${SSH_KEY} \
                        -o StrictHostKeyChecking=no \
                        ubuntu@${K8S_HOST} \
                        "kubectl rollout status \
                        deployment/my-frontend-mydeploy \
                        --timeout=180s"

                    echo "Kubernetes rollout successful!"
                '''
            }
        }

        stage('Verify Image') {
            steps {
                sh '''
                    echo "=============================================="
                    echo "Verifying Deployed Image"
                    echo "=============================================="

                    DEPLOYED_IMAGE=$(ssh -i ${SSH_KEY} \
                        -o StrictHostKeyChecking=no \
                        ubuntu@${K8S_HOST} \
                        "kubectl get deployment my-frontend-mydeploy \
                        -o=jsonpath='{.spec.template.spec.containers[0].image}'")

                    echo "Expected Image:"
                    echo "${IMAGE_NAME}:${IMAGE_TAG}"

                    echo "Deployed Image:"
                    echo "${DEPLOYED_IMAGE}"

                    if [ "${DEPLOYED_IMAGE}" != "${IMAGE_NAME}:${IMAGE_TAG}" ]; then
                        echo "ERROR: Image verification failed!"
                        exit 1
                    fi

                    echo "Image verification PASSED!"
                '''
            }
        }
    }

    post {
        success {
            echo '=============================================='
            echo '           PIPELINE SUCCESSFUL               '
            echo '=============================================='
            echo 'Docker build       : PASSED'
            echo 'BVT                : PASSED'
            echo 'Docker push        : PASSED'
            echo 'Helm deployment    : PASSED'
            echo 'K8s rollout        : PASSED'
            echo 'Image verification : PASSED'
            echo '=============================================='
        }

        failure {
            echo '=============================================='
            echo '             PIPELINE FAILED                 '
            echo '=============================================='
            echo 'Check the failed stage in Console Output.'
            echo '=============================================='
        }
    }
}
