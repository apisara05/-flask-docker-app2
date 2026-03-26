// =================================================================
// HELPER FUNCTION: ส่ง Notification ไปยัง n8n (ตามแนวทางของ Express Pipeline)
// =================================================================

def sendNotificationToN8n(String status, String stageName, String imageTag, String containerName, String hostPort) {
    // ต้องติดตั้ง HTTP Request Plugin และสร้าง Secret Text ชื่อ 'n8n-webhook'
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
    // ใช้ agent any เพราะ build จะทำงานบน Jenkins controller/agent (Linux)
    agent any

    // กันเช็คเอาต์ซ้ำซ้อน (ตามแนวทาง Express)
    options {
        skipDefaultCheckout(true)
    }

    // Environment variables
    environment {
        DOCKER_HUB_CREDENTIALS_ID = 'dockerhub'
        DOCKER_REPO               = "iamsamitdev/flask-docker-app"

        // จำลอง DEV/PROD บน Local
        DEV_APP_NAME              = "flask-app-dev"
        DEV_HOST_PORT             = "5001"
        PROD_APP_NAME             = "flask-app-prod"
        PROD_HOST_PORT            = "5000"
    }

    // Input parameters (Build & Deploy หรือ Rollback)
    parameters {
        choice(name: 'ACTION', choices: ['Build & Deploy', 'Rollback'], description: 'เลือก Action ที่ต้องการ')
        string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'สำหรับ Rollback: ใส่ Image Tag (เช่น Git Hash หรือ dev-123)')
        choice(name: 'ROLLBACK_TARGET', choices: ['dev', 'prod'], description: 'สำหรับ Rollback: เลือก Environment')
    }

    stages {
        stage('Checkout') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        stage('Install & Test') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                echo "Running tests inside a consistent Docker environment..."
                script {
                    docker.image('python:3.13-slim').inside {
                        sh '''
                            pip install --no-cache-dir -r requirements.txt
                            pytest -v --tb=short --junitxml=test-results.xml
                        '''
                    }
                }
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
                    def imageTag = (env.BRANCH_NAME == 'main') ? sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim() : "dev-${env.BUILD_NUMBER}"
                    env.IMAGE_TAG = imageTag

                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_HUB_CREDENTIALS_ID) {
                        echo "Building image: ${DOCKER_REPO}:${env.IMAGE_TAG}"
                        def customImage = docker.build("${DOCKER_REPO}:${env.IMAGE_TAG}")

                        echo "Pushing images to Docker Hub..."
                        customImage.push()
                        if (env.BRANCH_NAME == 'main') {
                            customImage.push('latest')
                        }
                    }
                }
            }
        }

        stage('Deploy to DEV (Local Docker)') {
            when {
                expression { params.ACTION == 'Build & Deploy' }
            }
            steps {
                script {
                    def deployCmd = """
                            echo "Deploying container ${DEV_APP_NAME} from latest image..."
                            docker pull ${DOCKER_REPO}:${env.IMAGE_TAG}
                            docker stop ${DEV_APP_NAME} || true
                            docker rm ${DEV_APP_NAME} || true
                            docker run -d --name ${DEV_APP_NAME} -p ${DEV_HOST_PORT}:5000 ${DOCKER_REPO}:${env.IMAGE_TAG}
                            docker ps --filter name=${DEV_APP_NAME} --format "table {{.Names}}\\t{{.Image}}\\t{{.Status}}"
                        """
                    sh deployCmd
                }
            }
        }


        stage('Approval for Production') {
            when {
                expression { params.ACTION == 'Build & Deploy' }
            }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Deploy image tag '${env.IMAGE_TAG}' to PRODUCTION (Local Docker on port ${PROD_HOST_PORT})?"
                }
            }
        }

        stage('Deploy to PRODUCTION (Local Docker)') {
            when {
                expression { params.ACTION == 'Build & Deploy' }
            }
            steps {
                script {
                    def deployCmd = """
                            echo "Deploying container ${PROD_APP_NAME} from latest image..."
                            docker pull ${DOCKER_REPO}:${env.IMAGE_TAG}
                            docker stop ${PROD_APP_NAME} || true
                            docker rm ${PROD_APP_NAME} || true
                            docker run -d --name ${PROD_APP_NAME} -p ${PROD_HOST_PORT}:5000 ${DOCKER_REPO}:${env.IMAGE_TAG}
                            docker ps --filter name=${PROD_APP_NAME} --format "table {{.Names}}\\t{{.Image}}\\t{{.Status}}"
                        """
                    sh deployCmd
                }
            }
        }


        stage('Execute Rollback') {
            when { expression { params.ACTION == 'Rollback' } }
            steps {
                script {
                    if (params.ROLLBACK_TAG.trim().isEmpty()) {
                        error "เมื่อเลือก Rollback กรุณาระบุ 'ROLLBACK_TAG'"
                    }

                    env.TARGET_APP_NAME  = (params.ROLLBACK_TARGET == 'dev') ? env.DEV_APP_NAME  : env.PROD_APP_NAME
                    env.TARGET_HOST_PORT = (params.ROLLBACK_TARGET == 'dev') ? env.DEV_HOST_PORT : env.PROD_HOST_PORT
                    def imageToDeploy = "${DOCKER_REPO}:${params.ROLLBACK_TAG.trim()}"

                    echo "ROLLING BACK ${params.ROLLBACK_TARGET.toUpperCase()} to image: ${imageToDeploy}"

                    sh """
                        docker pull ${imageToDeploy}
                        docker stop ${env.TARGET_APP_NAME} || true
                        docker rm ${env.TARGET_APP_NAME} || true
                        docker run -d --name ${env.TARGET_APP_NAME} -p ${env.TARGET_HOST_PORT}:5000 ${imageToDeploy}
                    """
                }
            }
        }

    }

    // Post actions
    post {
        always {
            script {
                if (params.ACTION == 'Build & Deploy') {
                    echo "Cleaning up Docker images on agent..."
                    try {
                        sh """
                            docker image rm -f ${DOCKER_REPO}:${env.IMAGE_TAG} || true
                            docker image rm -f ${DOCKER_REPO}:latest || true
                        """
                    } catch (err) {
                        echo "Could not clean up images, but continuing..."
                    }
                }
                echo "Cleaning up workspace..."
                cleanWs()
            }
        }
    }
}