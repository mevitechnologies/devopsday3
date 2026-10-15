
pipeline {
    agent any

    stages {
        stage('Checkout Git') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t mevi-html-site:1.0 .'
            }
        }

        stage('Deploy Website') {
            steps {
                sh '''
                    docker rm -f mevi-html-site 2>/dev/null || true
                    docker run -d \
                        --name mevi-html-site \
                        --restart unless-stopped \
                        -p 8082:80 \
                        mevi-html-site:1.0
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh 'docker ps'
                sh 'curl -f http://localhost:8082/'
            }
        }
    }
}
