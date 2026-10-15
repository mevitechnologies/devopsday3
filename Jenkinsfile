pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        APP_NAME = 'mevi-html-site'
        IMAGE_NAME = 'mevi-html-site:1.0'
        NETWORK_NAME = 'mevi-network'
        JENKINS_CONTAINER = 'jenkins'
        HOST_PORT = '8082'
        CONTAINER_PORT = '80'
    }

    stages {

        stage('1. Checkout GitHub') {
            steps {
                echo 'Checking out source code from GitHub'

                checkout scm

                sh '''
                    set -eu

                    echo "Project files:"
                    ls -la

                    test -f Dockerfile || {
                        echo "ERROR: Dockerfile not found in repository root"
                        exit 1
                    }

                    test -f index.html || {
                        echo "ERROR: index.html not found in repository root"
                        exit 1
                    }

                    test -f Jenkinsfile || {
                        echo "ERROR: Jenkinsfile not found"
                        exit 1
                    }

                    echo "Required files are present."
                '''
            }
        }

        stage('2. Check Docker') {
            steps {
                echo 'Checking Docker CLI and daemon'

                sh '''
                    set -eu
                    docker --version
                    docker info
                '''
            }
        }

        stage('3. Configure Docker Network') {
            steps {
                echo 'Creating the Docker network if it does not exist'

                sh '''
                    set -eu

                    if ! docker network inspect "$NETWORK_NAME" >/dev/null 2>&1; then
                        docker network create "$NETWORK_NAME"
                    fi

                    echo "Docker network is ready."
                    docker network ls
                '''
            }
        }

        stage('4. Build Website Image') {
            steps {
                echo 'Building the website image'

                sh '''
                    set -eu

                    docker build \
                        -t "$IMAGE_NAME" .

                    echo "Image built successfully."
                    docker image inspect "$IMAGE_NAME"
                '''
            }
        }

        stage('5. Remove Old Website') {
            steps {
                echo 'Removing the old website container if it exists'

                sh '''
                    docker rm -f "$APP_NAME" 2>/dev/null || true
                '''
            }
        }

        stage('6. Deploy Website') {
            steps {
                echo 'Starting the new website container'

                sh '''
                    set -eu

                    docker run -d \
                        --name "$APP_NAME" \
                        --network "$NETWORK_NAME" \
                        --restart unless-stopped \
                        -p "$HOST_PORT:$CONTAINER_PORT" \
                        "$IMAGE_NAME"

                    echo "Website container started."
                    docker ps -a --filter "name=^/${APP_NAME}$"
                '''
            }
        }

        stage('7. Verify Container') {
            steps {
                echo 'Checking container status'

                sh '''
                    set -eu

                    RUNNING=$(docker inspect \
                        --format='{{.State.Running}}' "$APP_NAME")

                    if [ "$RUNNING" != "true" ]; then
                        echo "ERROR: Website container is not running."
                        docker logs "$APP_NAME" || true
                        exit 1
                    fi

                    echo "SUCCESS: Website container is running."
                '''
            }
        }

        stage('8. Verify Docker Network') {
            steps {
                echo 'Connecting Jenkins to the network for container-to-container testing'

                sh '''
                    set -eu

                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' \
                        | grep -q "$APP_NAME"; then

                        echo "ERROR: Website is not attached to $NETWORK_NAME."
                        docker network inspect "$NETWORK_NAME"
                        exit 1
                    fi

                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' \
                        | grep -q "$JENKINS_CONTAINER"; then

                        echo "Connecting Jenkins container to $NETWORK_NAME"

                        docker network connect \
                            "$NETWORK_NAME" "$JENKINS_CONTAINER"
                    fi

                    echo "Docker network configuration checked."
                    docker network inspect "$NETWORK_NAME"
                '''
            }
        }

        stage('9. Test Website') {
            steps {
                echo 'Testing the website through the Docker network'

                sh '''
                    set -eu

                    SUCCESS=0

                    for i in $(seq 1 15); do
                        echo "HTTP test attempt $i of 15"

                        if curl -fsS \
                            --connect-timeout 3 \
                            --max-time 5 \
                            "http://$APP_NAME/" \
                            -o /tmp/mevi-site.html; then

                            SUCCESS=1
                            break
                        fi

                        sleep 2
                    done

                    if [ "$SUCCESS" -ne 1 ]; then
                        echo "ERROR: Website did not respond."
                        docker logs "$APP_NAME" || true
                        exit 1
                    fi

                    echo "Website responded successfully."

                    echo "Checking the returned HTML..."

                    if grep -qi "Mevi Technologies" /tmp/mevi-site.html; then
                        echo "SUCCESS: Mevi Technologies content found."
                    else
                        echo "ERROR: Expected text was not found."
                        echo "Actual HTML response:"
                        head -n 30 /tmp/mevi-site.html
                        exit 1
                    fi
                '''
            }
        }

        stage('10. Verify Host Port') {
            steps {
                echo 'Checking website access through the published host port'

                sh '''
                    set -eu

                    echo "Checking host port mapping:"
                    docker port "$APP_NAME"

                    echo "Checking container HTTP response:"
                    docker exec "$APP_NAME" \
                        wget -q -O - http://127.0.0.1/ \
                        > /tmp/mevi-host-test.html

                    grep -qi "Mevi Technologies" /tmp/mevi-host-test.html

                    echo "Container HTTP test passed."
                    echo "Open http://localhost:8082/ in your Mac browser."
                '''
            }
        }

        stage('11. Deployment Summary') {
            steps {
                echo 'Displaying final deployment details'

                sh '''
                    echo "===================================="
                    echo "DEPLOYMENT SUMMARY"
                    echo "===================================="

                    echo "Docker image:"
                    docker image ls "$IMAGE_NAME"

                    echo "Website container:"
                    docker ps -a --filter "name=^/${APP_NAME}$"

                    echo "Published ports:"
                    docker port "$APP_NAME"

                    echo "Website URL: http://localhost:8082/"
                    echo "===================================="
                '''
            }
        }
    }

    post {
        success {
            echo '''
            ==========================================
            PIPELINE SUCCESSFUL!
            ==========================================
            GitHub checkout completed.
            Docker image built successfully.
            Website container deployed.
            Website HTTP response verified.

            Open the website:
            http://localhost:8082/
            ==========================================
            '''
        }

        failure {
            echo '''
            ==========================================
            PIPELINE FAILED
            ==========================================
            Open Console Output and find the first
            failed command.

            Check Docker access, network connectivity,
            Dockerfile, index.html, and container logs.
            ==========================================
            '''

            sh '''
                echo "Container status:"
                docker ps -a || true

                echo "Website container logs:"
                docker logs "$APP_NAME" 2>&1 || true

                echo "Docker network:"
                docker network inspect "$NETWORK_NAME" 2>&1 || true
            '''
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}