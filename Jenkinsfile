// =================================================================
// HELPER FUNCTION: ส่ง Notification ไปยัง n8n
// =================================================================
def sendNotificationToN8n(String status, String stageName, String imageTag, String containerName, String hostPort) {
    script {
        withCredentials([string(credentialsId: 'n8n-webhook', variable: 'N8N_WEBHOOK_URL')]) {
            def payload = [
                project  : env.JOB_NAME,
                stage    : stageName,
                status   : status,
                build    : env.BUILD_NUMBER,
                image    : "${env.DOCKER_REPO}:${imageTag}",
                container: containerName,
                url      : "http://localhost:${hostPort}/",
                timestamp: new Date().format("yyyy-MM-dd'T'HH:mm:ssXXX")
            ]
            def body = groovy.json.JsonOutput.toJson(payload)
            try {
                httpRequest acceptType: 'APPLICATION_JSON',
                            contentType: 'APPLICATION_JSON',
                            httpMode: 'POST',
                            requestBody: body,
                            url: N8N_WEBHOOK_URL,
                            validResponseCodes: '200:299'
                echo "n8n webhook (${status}) sent successfully."
            } catch (err) {
                echo "Failed to send n8n webhook (${status}): ${err}"
            }
        }
    }
}

pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub-card1'
        DOCKER_REPO               = "apisara0508/flask-docker-app2"

        DEV_APP_NAME              = "flask-app-dev"
        DEV_HOST_PORT             = "5001"
        PROD_APP_NAME             = "flask-app-prod"
        PROD_HOST_PORT            = "5000"
    }

    parameters {
        choice(name: 'ACTION', choices: ['Build & Deploy', 'Rollback'], description: 'เลือก Action')
        string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'Tag สำหรับ rollback')
        choice(name: 'ROLLBACK_TARGET', choices: ['dev', 'prod'], description: 'เลือก environment')
    }

    stages {

        stage('Checkout') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                checkout scm
            }
        }

        stage('Install & Test') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                echo "Run test in Docker container..."
                sh '''
                    docker run --rm \
                    -v $(pwd):/app \
                    -w /app \
                    python:3.13-slim \
                    sh -c "
                        pip install --no-cache-dir -r requirements.txt &&
                        pytest -v --tb=short --junitxml=test-results.xml
                    "
                '''
            }
            post {
                always {
                    junit 'test-results.xml'
                }
            }
        }

        stage('Build & Push Docker Image') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                script {
                    def imageTag = (env.BRANCH_NAME == 'main') ?
                        sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim() :
                        "dev-${env.BUILD_NUMBER}"

                    env.IMAGE_TAG = imageTag

                    sh """
                        echo "Login Docker Hub..."
                        docker login -u \$DOCKER_USER -p \$DOCKER_PASS

                        echo "Build image..."
                        docker build -t ${DOCKER_REPO}:${env.IMAGE_TAG} .

                        echo "Push image..."
                        docker push ${DOCKER_REPO}:${env.IMAGE_TAG}

                        if [ "${env.BRANCH_NAME}" = "main" ]; then
                            docker tag ${DOCKER_REPO}:${env.IMAGE_TAG} ${DOCKER_REPO}:latest
                            docker push ${DOCKER_REPO}:latest
                        fi
                    """
                }
            }
        }

        stage('Deploy to DEV') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                sh """
                    docker pull ${DOCKER_REPO}:${env.IMAGE_TAG}
                    docker stop ${DEV_APP_NAME} || true
                    docker rm ${DEV_APP_NAME} || true
                    docker run -d --name ${DEV_APP_NAME} -p ${DEV_HOST_PORT}:5000 ${DOCKER_REPO}:${env.IMAGE_TAG}
                """
            }
        }

        stage('Approval for Production') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Deploy ${env.IMAGE_TAG} to PROD?"
                }
            }
        }

        stage('Deploy to PROD') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                sh """
                    docker pull ${DOCKER_REPO}:${env.IMAGE_TAG}
                    docker stop ${PROD_APP_NAME} || true
                    docker rm ${PROD_APP_NAME} || true
                    docker run -d --name ${PROD_APP_NAME} -p ${PROD_HOST_PORT}:5000 ${DOCKER_REPO}:${env.IMAGE_TAG}
                """
            }
        }

        stage('Rollback') {
            when { expression { params.ACTION == 'Rollback' } }
            steps {
                script {
                    if (params.ROLLBACK_TAG.trim().isEmpty()) {
                        error "กรุณาใส่ ROLLBACK_TAG"
                    }

                    def app = (params.ROLLBACK_TARGET == 'dev') ? DEV_APP_NAME : PROD_APP_NAME
                    def port = (params.ROLLBACK_TARGET == 'dev') ? DEV_HOST_PORT : PROD_HOST_PORT
                    def image = "${DOCKER_REPO}:${params.ROLLBACK_TAG}"

                    sh """
                        docker pull ${image}
                        docker stop ${app} || true
                        docker rm ${app} || true
                        docker run -d --name ${app} -p ${port}:5000 ${image}
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker system prune -f || true"
            cleanWs()
        }
    }
}