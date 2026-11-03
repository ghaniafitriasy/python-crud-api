pipeline {
    agent any

    environment {
        REGISTRY_NAME  = 'python-crud-api'
        CONTAINER_NAME = 'production-api-container'
        VOLUME_NAME    = 'production-api-data'
        NETWORK_NAME   = 'jenkins-net'
    }

    stages {
        stage('1. Checkout Code') {
            steps {
                echo 'Checking out source code from GitHub'
                checkout scm
            }
        }

        stage('2. Code Syntax Verification') {
            steps {
                // Jenkins container tidak punya Python, jadi cek sintaks
                // dilakukan lewat container python sementara.
                sh '''
                    docker run --rm -i python:3.11-slim \
                      python -c 'import sys; compile(sys.stdin.read(), "app.py", "exec"); print("Syntax OK")' < app.py
                '''
            }
        }

        stage('3. Build Docker Image') {
            steps {
                sh 'docker build -t ${REGISTRY_NAME}:${BUILD_NUMBER} .'
                sh 'docker tag ${REGISTRY_NAME}:${BUILD_NUMBER} ${REGISTRY_NAME}:latest'
            }
        }

        stage('4. Automated Deployment') {
            steps {
                sh '''
                    docker stop ${CONTAINER_NAME} || true
                    docker rm ${CONTAINER_NAME} || true
                    docker volume create ${VOLUME_NAME}
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      --network ${NETWORK_NAME} \
                      -p 5000:5000 \
                      -v ${VOLUME_NAME}:/var/lib/backend_data \
                      ${REGISTRY_NAME}:latest
                '''
            }
        }

        stage('5. Health Check') {
            steps {
                sh 'sleep 3'
                // Jenkins berada di jaringan jenkins-net, jadi API dipanggil lewat nama container.
                sh 'curl -fsS http://${CONTAINER_NAME}:5000/health'
                sh 'docker logs --tail 20 ${CONTAINER_NAME}'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
        }
        failure {
            echo 'Pipeline failed. Inspect Console Output.'
        }
    }
}
