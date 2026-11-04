pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'rawanm21/ivolve-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKER_CREDS = 'docker'
        K8S_NAMESPACE = 'ivolve'
        DEPLOYMENT = 'ivolveapp'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/rawanmahmoud659-hub/ivolve-app.git'
            }
        }

        stage('Build App') {
            steps {
                sh '''
                    npm install
                    npm test --if-present
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t ${DOCKER_IMAGE}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Scan Docker Image') {
            steps {
                sh '''
                    trivy image --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      --format table \
                      --output trivy-report.txt \
                      ${DOCKER_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CREDS}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set +x
                        export DOCKER_CONFIG="$(mktemp -d)"
                        trap 'docker logout >/dev/null 2>&1 || true; rm -rf "$DOCKER_CONFIG"' EXIT

                        echo "$DOCKER_PASS" | docker login \
                          -u "$DOCKER_USER" --password-stdin

                        docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Delete Local Image') {
            steps {
                sh '''
                    docker rmi ${DOCKER_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Update Deployment File') {
            steps {
                sh '''
                    sed -i \
                      "s|image: rawanm21/ivolve-app:.*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|" \
                      deployment.yaml

                    grep 'image:' deployment.yaml
                '''
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f deployment.yaml \
                      -n ${K8S_NAMESPACE}

                    kubectl rollout status \
                      deployment/${DEPLOYMENT} \
                      -n ${K8S_NAMESPACE} \
                      --timeout=180s
                '''
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
            archiveArtifacts artifacts: 'trivy-report.txt',
                             allowEmptyArchive: true
        }

        success {
            echo 'SUCCESS: Application deployed to Kubernetes!'
        }

        failure {
            echo 'FAILURE: Pipeline failed. Check Jenkins logs.'
        }
    }
}

