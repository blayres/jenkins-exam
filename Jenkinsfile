pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_USER = 'blayres'
        MOVIE_IMAGE = 'blayres/movie-service'
        CAST_IMAGE = 'blayres/cast-service'
    }

    stages {

        stage('Validate') {
            steps {
                sh 'docker compose config'
                sh 'helm lint ./charts'
            }
        }

        stage('Test') {
            steps {
                sh 'docker compose up -d --build'

                sh '''
                    echo "Waiting for Movie API..."
                    for i in {1..30}; do
                      if curl -sf http://localhost:8001/api/v1/movies/docs > /dev/null; then
                        break
                      fi
                      sleep 2
                    done

                    curl -f http://localhost:8001/api/v1/movies/docs > /dev/null
                    echo "Movie API OK"
                '''

                sh '''
                    echo "Waiting for Cast API..."
                    for i in {1..30}; do
                      if curl -sf http://localhost:8002/api/v1/casts/docs > /dev/null; then
                        break
                      fi
                      sleep 2
                    done

                    curl -f http://localhost:8002/api/v1/casts/docs > /dev/null
                    echo "Cast API OK"
                '''
            }

            post {
                always {
                    sh 'docker compose down || true'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -t ${MOVIE_IMAGE}:${BUILD_NUMBER} ./movie-service
                    docker build -t ${CAST_IMAGE}:${BUILD_NUMBER} ./cast-service
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'DOCKER_HUB_PASS',
                        variable: 'DOCKER_HUB_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_HUB_PASS" | docker login ${DOCKER_REGISTRY} \
                            -u ${DOCKER_USER} \
                            --password-stdin

                        docker push ${MOVIE_IMAGE}:${BUILD_NUMBER}
                        docker push ${CAST_IMAGE}:${BUILD_NUMBER}

                        docker logout ${DOCKER_REGISTRY}
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker compose down || true'
        }

        success {
            echo 'CI pipeline completed successfully.'
        }

        failure {
            echo 'CI pipeline failed.'
        }
    }
}
