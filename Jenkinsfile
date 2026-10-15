
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
        HEALTH_URL = 'http://localhost:8082/'
    }

    stages {

        // STAGE 1: CHECK OUT SOURCE CODE
        stage('1. Checkout GitHub') {
            steps {
                echo 'Downloading project source code from GitHub...'

                checkout scm

                sh '''
                    echo "Project files:"
                    ls -la

                    echo "Checking required files..."

                    test -f Dockerfile || {
                        echo "ERROR: Dockerfile not found!"
                        exit 1
                    }

                    test -f index.html || {
                        echo "ERROR: index.html not found!"
                        exit 1
                    }
                '''
            }
        }

        // STAGE 2: CHECK DOCKER
        stage('2. Check Docker') {
            steps {
                echo 'Checking Docker installation and daemon access...'

                sh '''
                    set -eu

                    docker --version
                    docker info
                '''
            }
        }

        // STAGE 3: CREATE NETWORK
        stage('3. Configure Docker Network') {
            steps {
                echo 'Creating the Docker network if required...'

                sh '''
                    set -eu

                    if ! docker network inspect "$NETWORK_NAME" >/dev/null 2>&1; then
                        docker network create "$NETWORK_NAME"
                    fi

                    echo "Docker network details:"
                    docker network inspect "$NETWORK_NAME"
                '''
            }
        }

        // STAGE 4: BUILD IMAGE
        stage('4. Build Website Image') {
            steps {
                echo 'Building the website image from the Dockerfile...'

                sh '''
                    set -eu

                    docker build \
                        --pull=false \
                        -t "$IMAGE_NAME" .
                '''

                echo 'Website image built successfully.'
            }
        }

        // STAGE 5: REMOVE OLD CONTAINER AND DEPLOY
        stage('5. Deploy Website') {
            steps {
                echo 'Removing the previous website container, if present...'

                sh '''
                    docker rm -f "$APP_NAME" 2>/dev/null || true
                '''

                echo 'Starting the new website container...'

                sh '''
                    set -eu

                    docker run -d \
                        --name "$APP_NAME" \
                        --network "$NETWORK_NAME" \
                        --restart unless-stopped \
                        -p "$HOST_PORT:$CONTAINER_PORT" \
                        "$IMAGE_NAME"
                '''
            }
        }

        // STAGE 6: VERIFY CONTAINER
        stage('6. Verify Container') {
            steps {
                echo 'Checking whether the website container is running...'

                sh '''
                    set -eu

                    docker ps --filter "name=^/${APP_NAME}$"

                    RUNNING=$(docker inspect \
                        --format='{{.State.Running}}' "$APP_NAME")

                    if [ "$RUNNING" != "true" ]; then
                        echo "ERROR: Website container is not running."
                        docker logs "$APP_NAME"
                        exit 1
                    fi

                    echo "Website container is running."
                '''
            }
        }

        // STAGE 7: VERIFY WEBSITE
        stage('7. Test Website') {
            steps {
                echo 'Testing the deployed website from the Jenkins container...'

                sh '''
                    set -eu

                    # The Jenkins container must share this network
                    # with the website container for this test.
                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' \
                        | grep -q "$APP_NAME"; then
                        echo "ERROR: Website container is not attached to the expected network."
                        docker network inspect "$NETWORK_NAME"
                        exit 1
                    fi

                    # Attach Jenkins to the network if necessary.
                    if ! docker network inspect "$NETWORK_NAME" \
                        --format='{{json .Containers}}' \
                        | grep -q "$JENKINS_CONTAINER"; then
                        echo "Connecting Jenkins to the Docker network..."
                        docker network connect \
                            "$NETWORK_NAME" "$JENKINS_CONTAINER"
                    fi

                    # Retry HTTP verification.
                    SUCCESS=0

                    for i in $(seq 1 15); do
                        echo "Website test attempt $i of 15..."

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
                        echo "ERROR: Website did not respond over the Docker network."
                        docker logs "$APP_NAME" || true
                        docker network inspect "$NETWORK_NAME"
                        exit 1
                    fi

                    echo "Website