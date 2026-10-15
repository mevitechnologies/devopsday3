
pipeline {
    agent any

    stages {

        stage('Checkout GitHub') {
            steps {
                echo 'Checking out source code from GitHub...'
                checkout scm
            }
        }

        stage('Check Docker') {
            steps {
                echo 'Checking Docker CLI and daemon access...'
                sh 'docker --version'
                sh 'docker ps'
            }
        }

        stage('Configure Docker Network') {
            steps {
                echo 'Creating shared network and connecting Jenkins...'
                sh '''
                    docker network inspect mevi-network >/dev/null 2>&1 \
                        || docker network create mevi-network

                    docker network connect mevi-network jenkins \
                        2>/dev/null || true
                '''
            }
        }

        stage('Build Website Image') {
            steps {
                echo 'Building the website Docker image...'
                sh 'docker build -t mevi-html-site:1.0 .'
            }
        }

        stage('Deploy Website') {
            steps {
                echo 'Removing old website container if present...'
                sh 'docker rm -f mevi-html-site 2>/dev/null || true'

                echo 'Starting website on the shared network...'
                sh '''
                    docker run -d \
                        --name mevi-html-site \
                        --network mevi-network \
                        --restart unless-stopped \
                        -p 8082:80 \
                        mevi-html-site:1.0
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Waiting for Nginx to start...'
                sh '''
                    for i in $(seq 1 15); do
                        if curl -fsS http://mevi-html-site/ -o /tmp/mevi-site.html; then
                            echo "Website is responding successfully!"
                            break
                        fi

                        if [ "$i" -eq 15 ]; then
                            echo "Website verification failed."
                            exit 1
                        fi

                        sleep 2
                    done
                '''

                echo 'Checking the website content...'
                sh "grep -i 'Mevi Technologies' /tmp/mevi-site.html"
            }
        }

        stage('Deployment Summary') {
            steps {
                echo 'Running container status check...'
                sh 'docker ps'
                echo 'Website URL: http://localhost:8082'
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Website deployed and verified!'
        }

        failure {
            echo 'FAILED: Check Console Output for the first error.'
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
