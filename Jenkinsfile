// Lab 8 - CD Pipeline: Checkout -> Docker Build -> Scan -> Push -> Deploy -> Verify
// Dang Duc Manh - DTC235180119
pipeline {
    agent any

    environment {
        DOCKER_USER     = 'manh28052005'   // tài khoản Docker Hub dùng cho Jenkins (credential docker-hub-credentials_1)
        DOCKER_IMAGE    = "${DOCKER_USER}/devops-lab8-app"
        DOCKER_TAG      = "${env.BUILD_NUMBER}"
        CONTAINER_NAME  = 'lab8-cd-production'
        APP_PORT        = '3002'   // 3000 đang dùng cho stack Docker Compose (Bước 2)
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'CD STEP 1/6: CHECKOUT'
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                echo 'CD STEP 2/6: BUILD IMAGE'
                sh '''
                    docker build \
                        -t ${DOCKER_IMAGE}:${DOCKER_TAG} \
                        -t ${DOCKER_IMAGE}:latest \
                        .
                    docker images ${DOCKER_IMAGE}
                '''
            }
        }

        stage('Docker Scan') {
            steps {
                echo 'CD STEP 3/6: SCAN IMAGE'
                sh '''
                    docker scout quickview ${DOCKER_IMAGE}:${DOCKER_TAG} || echo "Scout not available — skip"
                '''
            }
        }

        stage('Push to Registry') {
            steps {
                echo 'CD STEP 4/6: PUSH TO DOCKER HUB'
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials_1', usernameVariable: 'HUB_USER', passwordVariable: 'HUB_PASS')]) {
                    sh '''
                        echo "$HUB_PASS" | docker login -u "$HUB_USER" --password-stdin
                        docker push ${DOCKER_IMAGE}:${DOCKER_TAG}
                        docker push ${DOCKER_IMAGE}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy Container') {
            steps {
                echo 'CD STEP 5/6: DEPLOY CONTAINER'
                sh '''
                    # Stop & remove container cũ nếu có
                    docker stop ${CONTAINER_NAME} 2>/dev/null || true
                    docker rm ${CONTAINER_NAME} 2>/dev/null || true

                    # Deploy container mới
                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        -p ${APP_PORT}:3000 \
                        -e NODE_ENV=production \
                        --restart unless-stopped \
                        ${DOCKER_IMAGE}:${DOCKER_TAG}
                '''
            }
        }

        stage('Verify Deploy') {
            steps {
                echo 'CD STEP 6/6: VERIFY'
                sh '''
                    sleep 3
                    curl -f http://host.docker.internal:${APP_PORT}/health || exit 1
                    echo ""
                    curl http://host.docker.internal:${APP_PORT}/
                '''
            }
        }
    }

    post {
        success {
            echo "CD PIPELINE SUCCESS — Deployed ${DOCKER_IMAGE}:${DOCKER_TAG}"
        }
        failure {
            echo "CD PIPELINE FAILED — Check logs!"
        }
        always {
            cleanWs()
        }
    }
}
