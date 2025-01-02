pipeline {
    environment { 
        DOCKER_ID = "afarah1208"
        MOVIE_IMAGE = "movie-service"
        CAST_IMAGE = "cast-service"
        DOCKER_TAG = "v.${BUILD_ID}.0"
    }
    agent any 
    stages {
        stage('Docker Build - Movie Service') { 
            steps {
                script {
                    sh '''
                        docker rm -f movie-service || true
                        docker build -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG movie-service/.
                        sleep 6
                    '''
                }
            }
        }
        stage('Docker Build - Cast Service') { 
            steps {
                script {
                    sh '''
                        docker rm -f cast-service || true
                        docker build -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG cast-service/.
                        sleep 6
                    '''
                }
            }
        }
        stage('Docker Run - Movie Service') { 
            steps {
                script {
                    sh '''
                        docker run -d \
                            -e DB_HOST=localhost \
                            -e DB_NAME=movies_test \
                            -e DB_USER=test_user \
                            -e DB_PASSWORD=test_password \
                            -p 8001:8001 \
                            --name movie-service \
                            $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
                        sleep 10
                    '''
                }
            }
        }
        stage('Test Acceptance - Movie Service') { 
            steps {
                script {
                    sh '''
                        curl -f http://localhost:8001/health || echo "Health check failed for movie-service"
                        docker stop movie-service
                        docker rm movie-service
                    '''
                }
            }
        }
        stage('Docker Run - Cast Service') { 
            steps {
                script {
                    sh '''
                        docker run -d \
                            -e DB_HOST=localhost \
                            -e DB_NAME=casts_test \
                            -e DB_USER=test_user \
                            -e DB_PASSWORD=test_password \
                            -p 8002:8002 \
                            --name cast-service \
                            $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                        sleep 10
                    '''
                }
            }
        }
        stage('Test Acceptance - Cast Service') { 
            steps {
                script {
                    sh '''
                        curl -f http://localhost:8002/health || echo "Health check failed for cast-service"
                        docker stop cast-service
                        docker rm cast-service
                    '''
                }
            }
        }
        stage('Docker Push') { 
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS")
            }
            steps {
                script {
                    sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
                        docker push $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                    '''
                }
            }
        }
        stage('Deploy to Dev') {
            environment {
                KUBECONFIG = credentials("config")
                VALUES_FILE = "values-dev.yaml"
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace dev || kubectl create namespace dev
                        sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" ${VALUES_FILE}
                        helm upgrade --install my-app ./my-app-helm --values=${VALUES_FILE} --namespace dev
                    '''
                }
            }
        }
        stage('Deploy to QA') {
            environment {
                KUBECONFIG = credentials("config")
                VALUES_FILE = "values-qa.yaml"
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace qa || kubectl create namespace qa
                        sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" ${VALUES_FILE}
                        helm upgrade --install my-app ./my-app-helm --values=${VALUES_FILE} --namespace qa
                    '''
                }
            }
        }
        stage('Deploy to Staging') {
            environment {
                KUBECONFIG = credentials("config")
                VALUES_FILE = "values-staging.yaml"
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace staging || kubectl create namespace staging
                        sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" ${VALUES_FILE}
                        helm upgrade --install my-app ./my-app-helm --values=${VALUES_FILE} --namespace staging
                    '''
                }
            }
        }
        stage('Deploy to Prod') {
            environment {
                KUBECONFIG = credentials("config")
                VALUES_FILE = "values-prod.yaml"
            }
            steps {
                timeout(time: 15, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace prod || kubectl create namespace prod
                        sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" ${VALUES_FILE}
                        helm upgrade --install my-app ./my-app-helm --values=${VALUES_FILE} --namespace prod
                    '''
                }
            }
        }
    }
}
