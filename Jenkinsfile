pipeline {
    agent any

    environment {
        IMAGE_NAME    = 'backend-app'
        REGISTRY_REPO = 'hamzanajj/backend-app'   // your private Docker Hub repo
        NETWORK       = 'devops-net'
        DB_NAME       = 'test_db'
        DB_PASSWORD   = 'root'
    }

    stages {

        stage('Checkout Git') {
            steps {
                checkout scm
            }
        }

        stage('Build & Test Maven') {
            steps {
                dir('backend') {
                    sh 'mvn clean test'
                    sh 'mvn package -DskipTests'
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir('backend') {
                    sh 'docker build -t $IMAGE_NAME:latest .'
                }
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred',
                                 usernameVariable: 'DOCKER_USER',
                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag $IMAGE_NAME:latest $REGISTRY_REPO:latest
                        docker push $REGISTRY_REPO:latest
                    '''
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    docker network create $NETWORK || true

                    if [ "$(docker ps -aq -f name=^mysql$)" ]; then
                        echo "MySQL container already exists - starting it"
                        docker network connect $NETWORK mysql || true
                        docker start mysql
                    else
                        docker run -d --name mysql \
                          --network $NETWORK \
                          -p 3306:3306 \
                          -e MYSQL_ROOT_PASSWORD=$DB_PASSWORD \
                          -e MYSQL_DATABASE=$DB_NAME \
                          --restart always \
                          mysql:8
                    fi

                    echo "Waiting for MySQL to be ready..."
                    for i in $(seq 1 30); do
                        if docker exec mysql mysqladmin ping -uroot -p$DB_PASSWORD --silent; then
                            echo "MySQL is ready"
                            break
                        fi
                        sleep 2
                    done
                '''
            }
        }

        stage('Deploy backend-app') {
            steps {
                sh '''
                    docker stop backend-app || true
                    docker rm backend-app || true

                    docker pull $REGISTRY_REPO:latest

                    docker run -d --name backend-app \
                      --network $NETWORK \
                      -p 8089:8080 \
                      -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/$DB_NAME?createDatabaseIfNotExist=true" \
                      -e SPRING_DATASOURCE_USERNAME=root \
                      -e SPRING_DATASOURCE_PASSWORD=$DB_PASSWORD \
                      --restart on-failure \
                      $REGISTRY_REPO:latest
                '''
            }
        }

        stage('Verification') {
            steps {
                sh '''
                    docker ps
                    echo "Waiting for backend-app to start..."
                    for i in $(seq 1 24); do
                        if docker logs backend-app 2>&1 | grep -q "Started BackendApplication"; then
                            break
                        fi
                        sleep 5
                    done
                    docker logs backend-app
                    docker logs backend-app 2>&1 | grep "Started BackendApplication"
                '''
            }
        }
    }

    post {
        always  { sh 'docker logout || true' }
        success { echo 'CD pipeline finished: SUCCESS' }
        failure { echo 'CD pipeline FAILED - check Console Output' }
    }
}
