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
                    success=false

                    for i in $(seq 1 30); do
                        if curl -sf http://localhost:8001/api/v1/movies/docs > /dev/null; then
                            success=true
                            echo "Movie API OK"
                            break
                        fi

                        echo "Movie API not ready yet (attempt $i/30)"
                        sleep 2
                    done

                    if [ "$success" != "true" ]; then
                        echo "Movie API failed to become ready"
                        exit 1
                    fi
                '''

                sh '''
                    echo "Waiting for Cast API..."
                    success=false

                    for i in $(seq 1 30); do
                        if curl -sf http://localhost:8002/api/v1/casts/docs > /dev/null; then
                            success=true
                            echo "Cast API OK"
                            break
                        fi

                        echo "Cast API not ready yet (attempt $i/30)"
                        sleep 2
                    done

                    if [ "$success" != "true" ]; then
                        echo "Cast API failed to become ready"
                        exit 1
                    fi
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

        stage('Deploy DEV') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        helm upgrade --install jenkins-exam-dev ./charts \
                          --namespace dev \
                          --create-namespace \
                          --set movie.image.tag=${BUILD_NUMBER} \
                          --set cast.image.tag=${BUILD_NUMBER}
                    '''
                }
            }
        }

	stage('Deploy QA') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'config',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    sh '''
                        export KUBECONFIG="$KUBECONFIG_FILE"

                        helm upgrade --install jenkins-exam-qa ./charts \
                          --namespace qa \
                          --create-namespace \
                          --set movie.image.tag=${BUILD_NUMBER} \
                          --set cast.image.tag=${BUILD_NUMBER}
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
            echo 'CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'CI/CD pipeline failed.'
        }
    }
}
