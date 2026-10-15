
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
                echo 'Checking Docker access...'
                sh 'docker --version'
                sh 'docker ps'
            }
        }

        stage('Create Docker Network') {
            steps {
                echo 'Creating shared Docker network...'
                sh '''
                    docker network inspect mevi-network >/dev/null 2>&1 \
                    || docker network create mevi-network
                '''
            }
        }

        stage('Build Website Image') {
            steps {
                echo 'Building the Nginx website image...'
                sh 'docker build -t mevi-html-site:1.0 .'
            }
        }

        stage('Deploy Website') {
            steps {
                echo 'Deploying website container...'
                sh '''
                    docker rm -f mevi-html-site 2>/dev/null || true

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
                echo 'Checking website from Jenkins...'
                sh 'curl --retry 5 --retry-delay 2 --retry-connrefused -f http://mevi-html-site/'
            }
        }

        stage('Deployment Summary') {
            steps {
                echo 'Website deployment completed!'
                sh 'docker ps'
                echo 'Open the website at http://localhost:8082'
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: Website deployed and verified!'
        }

        failure {
            echo 'FAILED: Check Console Output for the error.'
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
