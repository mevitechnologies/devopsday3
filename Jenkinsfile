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
                        echo "ERROR: Dockerfile not found"
                        exit 1
                    }

                    test -f index.html || {
                        echo "ERROR: index.html not found"
                        exit 1
                    }

                    test -f Jenkinsfile || {
                        echo "ERROR: Jenkinsfile not found"
                        exit 1
                    }

                    echo "All required files are present."
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
                echo 'Creating Docker network if necessary'

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
                echo 'Building website image'

                sh '''
                    set -eu

                    docker build -t "$IMAGE_NAME" .

                    echo "Website image built successfully."
                    docker image ls "$IMAGE_NAME"
                '''
            }
        }

        stage('5. Remove Old Website') {
            steps {
                echo 'Removing old website container if it exists'

                sh '''
                    docker rm -f "$APP_NAME" 2>/dev/null || true
                '''
            }
        }

        stage('6. Deploy Website') {
            steps {
                echo 'Deploying website container'

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
                echo 'Checking website container status'

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
                echo 'Checking Docker network connectivity'

                sh '''
                    set -eu

                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' \
                        | grep -q "$APP_NAME"; then

                        echo "ERROR: Website is not connected to the network."
                        docker network inspect "$NETWORK_NAME"
                        exit 1
                    fi

                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' \
                        | grep -q "$JENKINS_CONTAINER"; then

                        echo "Connecting Jenkins to the network..."

                        docker network connect \
                            "$NETWORK_NAME" "$JENKINS_CONTAINER"
                    fi

                    echo "Jenkins and website network configuration verified."
                    docker network inspect "$NETWORK_NAME"
                '''
            }
        }

        stage('9. Test Website') {
            steps {
                echo 'Testing website HTTP response and content'

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

                    echo "SUCCESS: Website returned an HTTP response."

                    if grep -qi "Wisdom of Indian Saints" /tmp/mevi-site.html; then
                        echo "SUCCESS: Expected website title was found."
                    else
                        echo "ERROR: Expected website title was not found."
                        echo "Actual page title:"
                        grep -i "<title>" /tmp/mevi-site.html || true
                        exit 1
                    fi
                '''
            }
        }

        stage('10. Verify Published Port') {
            steps {
                echo 'Verifying website port mapping'

                sh '''
                    set -eu

                    echo "Published website ports:"
                    docker port "$APP_NAME"

                    echo "Checking Nginx response inside the website container..."

                    docker exec "$APP_NAME" \
                        wget -q -O - http://127.0.0.1/ \
                        > /tmp/mevi-container-page.html

                    if grep -qi "Wisdom of Indian Saints" /tmp/mevi-container-page.html; then
                        echo "SUCCESS: Website responds inside the container."
                    else
                        echo "ERROR: Website content verification failed."
                        exit 1
                    fi

                    echo "Host port: 8082"
                    echo "Website URL: http://localhost:8082/"
                '''
            }
        }

        stage('11. Deployment Summary') {
            steps {
                echo 'Displaying deployment summary'

                sh '''
                    echo "========================================"
                    echo "DEPLOYMENT SUMMARY"
                    echo "========================================"

                    echo "Docker image:"
                    docker image ls "$IMAGE_NAME"

                    echo "Website container:"
                    docker ps --filter "name=^/${APP_NAME}$"

                    echo "Published ports:"
                    docker port "$APP_NAME"

                    echo "Website URL: http://localhost:8082/"

                    echo "========================================"
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
            GitHub checkout: SUCCESS
            Docker check: SUCCESS
            Image build: SUCCESS
            Website deployment: SUCCESS
            HTTP verification: SUCCESS
            Content verification: SUCCESS

            Open your website:
            http://localhost:8082/
            ==========================================
            '''
        }

        failure {
            echo '''
            ==========================================
            PIPELINE FAILED
            ==========================================
            Open Console Output and locate the first
            failed command.

            Check Docker access, file names,
            container status, network connectivity,
            and the expected website title.
            ==========================================
            '''

            sh '''
                echo "All containers:"
                docker ps -a || true

                echo "Website container logs:"
                docker logs "$APP_NAME" 2>&1 || true

                echo "Docker network details:"
                docker network inspect "$NETWORK_NAME" 2>&1 || true
            '''
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}