pipeline {
    agent any

    environment {
        IMAGE_NAME = 'flask-hostname-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        DOCKERHUB_REPO = 'mathieu700/flask-hostname-app'
        YOUR_NAME = 'Matt'
        MAX_IMAGE_SIZE_MB = '250'
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
                    mkdir -p reports metadata

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
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_REPO}:${IMAGE_TAG}
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${DOCKERHUB_REPO}:latest
                '''
            }
        }

        stage('Collect metadata') {
            steps {
                sh '''
                    mkdir -p metadata

                    docker image inspect ${IMAGE_NAME}:${IMAGE_TAG} > metadata/image-inspect.json

                    IMAGE_SIZE_BYTES=$(docker image inspect ${IMAGE_NAME}:${IMAGE_TAG} --format='{{.Size}}')
                    IMAGE_SIZE_MB=$((IMAGE_SIZE_BYTES / 1024 / 1024))

                    echo "IMAGE_NAME=${IMAGE_NAME}" > metadata/build-metadata.txt
                    echo "IMAGE_TAG=${IMAGE_TAG}" >> metadata/build-metadata.txt
                    echo "DOCKERHUB_REPO=${DOCKERHUB_REPO}" >> metadata/build-metadata.txt
                    echo "IMAGE_SIZE_BYTES=${IMAGE_SIZE_BYTES}" >> metadata/build-metadata.txt
                    echo "IMAGE_SIZE_MB=${IMAGE_SIZE_MB}" >> metadata/build-metadata.txt
                    echo "GIT_COMMIT=$(git rev-parse HEAD || true)" >> metadata/build-metadata.txt
                    echo "BUILD_NUMBER=${BUILD_NUMBER}" >> metadata/build-metadata.txt
                    echo "BUILD_URL=${BUILD_URL}" >> metadata/build-metadata.txt
                '''
            }
        }

        stage('Gate image size') {
            steps {
                sh '''
                    IMAGE_SIZE_BYTES=$(docker image inspect ${IMAGE_NAME}:${IMAGE_TAG} --format='{{.Size}}')
                    IMAGE_SIZE_MB=$((IMAGE_SIZE_BYTES / 1024 / 1024))

                    echo "Image size: ${IMAGE_SIZE_MB} MB"
                    echo "Max allowed: ${MAX_IMAGE_SIZE_MB} MB"

                    if [ "$IMAGE_SIZE_MB" -gt "$MAX_IMAGE_SIZE_MB" ]; then
                      echo "Image is too large"
                      exit 1
                    fi
                '''
            }
        }

        stage('Optional Slim build') {
            steps {
                sh '''
                    mkdir -p metadata

                    if command -v slim >/dev/null 2>&1; then
                        echo "SlimToolkit found. Running slim build..." > metadata/docker-images-after-slim.txt

                        slim build \
                          --target ${IMAGE_NAME}:${IMAGE_TAG} \
                          --tag ${IMAGE_NAME}:${IMAGE_TAG}-slim || true

                        echo "" >> metadata/docker-images-after-slim.txt
                        docker images | grep ${IMAGE_NAME} >> metadata/docker-images-after-slim.txt || true
                    else
                        echo "SlimToolkit not installed on Jenkins controller." > metadata/docker-images-after-slim.txt
                        echo "Optional Slim build skipped." >> metadata/docker-images-after-slim.txt
                    fi
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

        stage('Manual release approval') {
            steps {
                script {
                    def approval = timeout(time: 10, unit: 'MINUTES') {
                        input(
                            message: """
Approve deployment?

Release details:
- Build Number: ${BUILD_NUMBER}
- Image: ${DOCKERHUB_REPO}:${IMAGE_TAG}
- Image Size Gate: Passed
- Trivy Filesystem Scan: Completed
- Trivy Image Scan: Passed
- Docker Slim Stage: Optional/Recorded
- Metadata: Archived after build

Approve only if the image scan and metadata are acceptable.
                            """,
                            ok: 'Approve deployment',
                            submitterParameter: 'APPROVER'
                        )
                    }

                    sh '''
                        mkdir -p metadata
                    '''

                    writeFile file: 'metadata/approval-metadata.txt', text: """APPROVED_BY=${approval}
APPROVED_IMAGE=${DOCKERHUB_REPO}:${IMAGE_TAG}
APPROVED_BUILD=${BUILD_NUMBER}
"""

                    sh '''
                        echo "APPROVAL_TIME_UTC=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> metadata/approval-metadata.txt
                    '''
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    sh '''
                        echo "$DOCKERHUB_TOKEN" | docker login -u "$DOCKERHUB_USER" --password-stdin

                        docker push ${DOCKERHUB_REPO}:${IMAGE_TAG}
                        docker push ${DOCKERHUB_REPO}:latest

                        docker logout
                    '''
                }
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
            archiveArtifacts artifacts: 'reports/*.txt, metadata/*', allowEmptyArchive: true
        }
    }
}
