
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Verify Project Files') {
            steps {
                sh '''
                    set -eu
                    test -f docker-compose.yml
                    test -f backend/Dockerfile
                    test -f frontend/Dockerfile
                    echo "JobTrack project files verified."
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -eu
                    docker compose build
                '''
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    set -eu
                    docker compose up -d mysql

                    echo "Waiting for MySQL to become healthy..."
                    for i in $(seq 1 30); do
                        STATUS=$(docker inspect \
                          --format='{{.State.Health.Status}}' jobtrack_mysql 2>/dev/null || true)

                        if [ "$STATUS" = "healthy" ]; then
                            echo "MySQL is healthy."
                            exit 0
                        fi

                        sleep 5
                    done

                    echo "MySQL did not become healthy."
                    docker compose logs --tail=80 mysql
                    exit 1
                '''
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    set -eu
                    docker compose up -d --build
                    sleep 10
                    docker compose ps
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                    set -eu

                    echo "Checking backend..."
                    curl --retry 5 --retry-delay 3 --retry-connrefused \
                         --fail http://localhost:8000/

                    echo ""
                    echo "Checking frontend..."
                    curl --retry 5 --retry-delay 3 --retry-connrefused \
                         --fail http://localhost:3000/

                    echo ""
                    echo "JobTrack deployment successful."
                '''
            }
        }
    }

    post {
        success {
            echo 'JobTrack deployment completed successfully.'
        }

        failure {
            sh '''
                docker compose ps || true
                docker compose logs --tail=60 backend frontend mysql || true
            '''
            echo 'Deployment failed. Check the console output.'
        }
    }
}
