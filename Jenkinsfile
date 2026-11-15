pipeline {
    agent { label 'docker-agent' }
    
    environment {
        DOCKER_HUB_USER = 'mushkatyxxx'
        IMAGE_PHP = 'mushkatyxxx/crudphp'
        IMAGE_MYSQL = 'mushkatyxxx/crudmysql'
        STACK_NAME = 'app'
        CANARY_STACK_NAME = 'app-canary'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
            }
        }

        stage('Build Images') {
            steps {
                echo 'Building Docker images...'
                sh 'docker build -t ${IMAGE_PHP}:${BUILD_NUMBER} ./php'
                sh 'docker build -t ${IMAGE_MYSQL}:${BUILD_NUMBER} ./mysql'
            }
        }

        stage('Push Images') {
            steps {
                echo 'Pushing images to Docker Hub...'
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker push ${IMAGE_PHP}:${BUILD_NUMBER}'
                    sh 'docker push ${IMAGE_MYSQL}:${BUILD_NUMBER}'
                }
            }
        }

        stage('Deploy Canary') {
            steps {
                echo 'Deploying canary stack...'
                sh 'docker stack deploy -c docker-compose_canary.yaml ${CANARY_STACK_NAME}'
                echo 'Waiting for canary to start...'
                sh 'sleep 15'
            }
        }

        stage('Test Canary') {
            steps {
                echo 'Testing canary deployment on port 8081...'
                // Простая проверка, что порт 8081 отвечает
                sh 'curl -f http://localhost:8081 || exit 1'
            }
        }

        stage('Promote to Production') {
            steps {
                echo 'Promoting canary to production...'
                // Обновляем основной стек новыми образами
                sh 'docker service update --image ${IMAGE_PHP}:${BUILD_NUMBER} ${STACK_NAME}_web-server'
                sh 'docker service update --image ${IMAGE_MYSQL}:${BUILD_NUMBER} ${STACK_NAME}_db'
                echo 'Waiting for production update...'
                sh 'sleep 10'
            }
        }

        stage('Cleanup Canary') {
            steps {
                echo 'Removing canary stack...'
                sh 'docker stack rm ${CANARY_STACK_NAME}'
            }
        }
    }

    post {
        always {
            echo 'Pipeline finished.'
        }
        failure {
            echo 'Pipeline failed! Rolling back or keeping stable version.'
            sh 'docker stack rm ${CANARY_STACK_NAME} || true'
        }
    }
}
