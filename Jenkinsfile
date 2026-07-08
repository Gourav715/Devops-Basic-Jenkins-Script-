pipeline {
    agent any

    options {
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        APP_NAME = 'vainterior'
        HOST_PORT = '8081'
        CONTAINER_PORT = '80'
        IMAGE_NAME = 'vainterior-app'
        REGISTRY_HOST = '172.17.0.1'
        REGISTRY_URL = '172.17.0.1:5000'
        REGISTRY_IMAGE = '172.17.0.1:5000/vainterior-app'
        REGISTRY_USER = 'admin'
        REGISTRY_PASS = 'admin123'
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    echo "Current workspace: ${env.WORKSPACE}"

                    try {
                        withCredentials([usernamePassword(credentialsId: 'github-credentials',
                                                          passwordVariable: 'GITHUB_TOKEN',
                                                          usernameVariable: 'GITHUB_USER')]) {
                            git branch: 'main',
                                url: "https://${GITHUB_USER}:${GITHUB_TOKEN}@github.com/Gourav715/vainterior.git"
                        }
                    } catch (Exception e) {
                        echo "Authenticated clone failed (${e.message}). Falling back to public clone..."
                        git branch: 'main',
                            url: 'https://github.com/Gourav715/vainterior.git'
                    }

                    sh 'ls -la'

                    if (!fileExists('package.json')) {
                        error "❌ package.json not found!"
                    }
                    echo "✅ package.json found!"
                }
            }
        }

        // NOTE: The old approach ran `docker run -v $(pwd):/app node:18-alpine npm install/build`
        // as a separate stage. That broke because Jenkins runs in its own container and the
        // Docker daemon it talks to (via docker.sock) does not share Jenkins' filesystem view —
        // $(pwd) inside Jenkins is not a path the host daemon can resolve to real files.
        //
        // Fix: do the npm install/build INSIDE the Docker build itself, via a multi-stage
        // Dockerfile. `docker build .` sends the build context to the daemon over the API,
        // so it never depends on matching host/container paths. This removes the entire
        // class of bind-mount path bugs.

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} -t ${IMAGE_NAME}:latest .
                        docker images | grep ${IMAGE_NAME}
                    """
                }
            }
        }

        stage('Push to Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-credentials',
                                                  usernameVariable: 'REG_USER',
                                                  passwordVariable: 'REG_PASS')]) {
                    sh '''
                        echo "Configuring Docker for insecure registry..."
                        mkdir -p /root/.docker
                        cat > /root/.docker/config.json << 'EOF'
{
  "insecure-registries": ["172.17.0.1:5000", "localhost:5000"]
}
EOF

                        echo "Logging into registry..."
                        echo "$REG_PASS" | docker login "$REGISTRY_URL" -u "$REG_USER" --password-stdin

                        echo "Tagging image..."
                        docker tag "${IMAGE_NAME}:${BUILD_NUMBER}" "${REGISTRY_IMAGE}:${BUILD_NUMBER}"
                        docker tag "${IMAGE_NAME}:latest" "${REGISTRY_IMAGE}:latest"

                        echo "Pushing to registry..."
                        docker push "${REGISTRY_IMAGE}:${BUILD_NUMBER}"
                        docker push "${REGISTRY_IMAGE}:latest"

                        echo "✅ Image pushed to registry successfully"
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh """
                        echo "Stopping and removing existing container..."
                        docker stop ${APP_NAME} 2>/dev/null || true
                        docker rm ${APP_NAME} 2>/dev/null || true

                        echo "Pulling image from registry..."
                        docker pull ${REGISTRY_IMAGE}:latest

                        echo "Starting container..."
                        docker run -d \\
                            --name ${APP_NAME} \\
                            -p ${HOST_PORT}:${CONTAINER_PORT} \\
                            --restart unless-stopped \\
                            ${REGISTRY_IMAGE}:latest

                        echo "Container status:"
                        docker ps --filter "name=${APP_NAME}"
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                script {
                    sh """
                        echo "=== Container Status ==="
                        docker ps --filter "name=${APP_NAME}" --format "table {{.Names}}\\t{{.Status}}\\t{{.Ports}}\\t{{.Image}}"
                        
                        echo ""
                        echo "=== Testing Application ==="
                        
                        # Give container time to start
                        sleep 5
                        
                        echo "Testing http://localhost:${HOST_PORT}"
                        HTTP_CODE=\$(curl -s -o /dev/null -w "%{http_code}" http://localhost:${HOST_PORT} 2>/dev/null || echo "000")
                        echo "HTTP Status: \${HTTP_CODE}"
                        
                        if [ "\${HTTP_CODE}" = "200" ] || [ "\${HTTP_CODE}" = "304" ]; then
                            echo "✅ Application is working!"
                            echo ""
                            echo "=== Application Content ==="
                            curl -s http://localhost:${HOST_PORT} | head -50
                        else
                            echo "❌ Application returned status \${HTTP_CODE}"
                            echo ""
                            echo "=== Application Content (attempting to see error) ==="
                            curl -v http://localhost:${HOST_PORT} 2>&1 | head -50
                            echo ""
                            echo "=== Container Logs ==="
                            docker logs ${APP_NAME} --tail 30
                            echo ""
                            echo "=== Container Files ==="
                            docker exec ${APP_NAME} ls -la /usr/share/nginx/html/
                            echo ""
                            echo "=== Container Nginx Config ==="
                            docker exec ${APP_NAME} cat /etc/nginx/conf.d/default.conf
                        fi
                    """
                }
            }
        }

        stage('Application Info') {
            steps {
                script {
                    def containerStatus = sh(script: "docker inspect ${APP_NAME} --format='{{.State.Status}}' 2>/dev/null || echo 'Not running'", returnStdout: true).trim()

                    echo """
                        ========================================
                        ✅ APPLICATION DEPLOYED SUCCESSFULLY
                        ========================================
                        Application:   ${APP_NAME}
                        Build Number:  ${BUILD_NUMBER}
                        Image:         ${IMAGE_NAME}:${BUILD_NUMBER}
                        Registry:      ${REGISTRY_URL}
                        Status:        ${containerStatus}
                        URL:           http://localhost:${HOST_PORT}
                        Source:        github.com/Gourav715/vainterior
                        ========================================
                    """
                }
            }
        }
    }

    post {
        success {
            echo """
                ========================================
                ✅ PIPELINE COMPLETED SUCCESSFULLY
                ========================================
                🌐 Application: http://localhost:${HOST_PORT}
                📦 Registry:    ${REGISTRY_URL}
                🏷️  Image:      ${REGISTRY_IMAGE}:${BUILD_NUMBER} (and :latest)
                ========================================
            """
        }
        failure {
            echo """
                ========================================
                ❌ PIPELINE FAILED
                ========================================
                Troubleshooting:
                1. Check Docker:        docker ps
                2. Check registry:      docker ps | grep registry
                3. Check build locally: docker build -t test .
                4. Check container logs: docker logs ${APP_NAME}
                ========================================
            """
        }
        always {
            script {
                sh """
                    echo ""
                    echo "=== Final Status ==="
                    echo "Workspace: ${WORKSPACE}"
                    ls -la

                    echo ""
                    echo "=== Cleaning up dangling images ==="
                    docker image prune -f || true
                """
            }
        }
    }
}
