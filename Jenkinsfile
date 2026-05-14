pipeline {
    agent any

    environment {
        IMAGE_NAME = 'flask-hostname-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        YOUR_NAME = 'Matt'
    }

    stages {
        stage('Clean up containers') {
            steps {
                sh '''
                    docker rm -f flask-app || true
                    docker rm -f nginx-proxy || true
                '''
            }
        }

        stage('Set up network') {
            steps {
                sh '''
                    docker network create app-network || true
                '''
            }
        }

        stage('Filesystem security scan') {
            steps {
                sh '''
                    mkdir -p reports

                    trivy fs \
                      --scanners vuln,secret,misconfig \
                      --format table \
                      --output reports/trivy-fs-report.txt \
                      .
                '''
            }
        }

        stage('Build image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Image security scan') {
            steps {
                sh '''
                    mkdir -p reports

                    trivy image \
                      --format table \
                      --output reports/trivy-image-report.txt \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Gate image scan') {
            steps {
                sh '''
                    trivy image \
                      --severity CRITICAL,HIGH \
                      --exit-code 1 \
                      --ignore-unfixed \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Run containers') {
            steps {
                sh '''
                    docker run -d \
                      --name flask-app \
                      --network app-network \
                      -e YOUR_NAME="${YOUR_NAME}" \
                      ${IMAGE_NAME}:${IMAGE_TAG}

                    docker run -d \
                      --name nginx-proxy \
                      --network app-network \
                      -p 80:80 \
                      -v "$WORKSPACE/nginx.conf:/etc/nginx/nginx.conf:ro" \
                      nginx
                '''
            }
        }

        stage('Run Python tests') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate

                    pip install --upgrade pip
                    pip install requests

                    python test_app.py
                '''
            }
        }

        stage('Test with curl') {
            steps {
                sh '''
                    docker ps
                    curl http://localhost
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'reports/*.txt', allowEmptyArchive: true
        }
    }
}
