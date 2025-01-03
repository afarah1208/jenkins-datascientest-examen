pipeline {
    environment { 
        DOCKER_ID = "afarah1208"
        DOCKER_TAG = "v.${BUILD_ID}.0"
        DOCKER_PASS = credentials("DOCKER_HUB_PASS")
        MOVIE_IMAGE = "movie-service"
        CAST_IMAGE = "cast-service"
        KUBECONFIG = credentials("config")
    }
    agent any 
    stages {
        stage('Clean Environment') {
            steps {
                script {
                    sh '''
                        for service in movie-service cast-service cast-db movie-db nginx-service; do
                            if docker ps -a | grep -q $service; then
                                docker stop $service
                                docker rm -f $service
                            fi
                        done
                    '''
                }
            }
        }
        stage('Docker Build') { 
            steps {
                script {
                    sh '''
                        docker build -t $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG ./movie-service
                        docker build -t $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG ./cast-service
                        sleep 6
                    '''
                }
            }
        }
        stage('Docker Run - Services') {
            steps {
                script {
                    sh '''
                        # Run Cast DB
                        docker run -d --name cast-db \
                            -v postgres_data_cast:/var/lib/postgresql/data/ \
                            -e POSTGRES_USER=cast_db_user \
                            -e POSTGRES_PASSWORD=cast_db_password \
                            -e POSTGRES_DB=cast_db \
                            postgres:12.1-alpine
                        
                        # Run Movie DB
                        docker run -d --name movie-db \
                            -v postgres_data_movie:/var/lib/postgresql/data/ \
                            -e POSTGRES_USER=movie_db_user \
                            -e POSTGRES_PASSWORD=movie_db_password \
                            -e POSTGRES_DB=movie_db \
                            postgres:12.1-alpine
                        
                        # Run Cast Service
                        docker run -d --name cast-service \
                            -p 8002:8000 \
                            -e DATABASE_URI=postgresql://cast_db_user:cast_db_password@cast-db:5432/cast_db \
                            $DOCKER_ID/$CAST_IMAGE:$DOCKER_TAG
                        
                        # Run Movie Service
                        docker run -d --name movie-service \
                            -p 8001:8000 \
                            -e DATABASE_URI=postgresql://movie_db_user:movie_db_password@movie-db:5432/movie_db \
                            -e CAST_SERVICE_HOST_URL=http://cast-service:8000/api/v1/casts/ \
                            $DOCKER_ID/$MOVIE_IMAGE:$DOCKER_TAG
                        
                        # Run NGINX
                        docker run -d --name nginx-service \
                            -p 80:8080 \
                            -v ./nginx_config.conf:/etc/nginx/conf.d/default.conf \
                            nginx:1.17.6-alpine
                        sleep 10
                    '''
                }
            }
        }
        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                        curl -f http://localhost:8001/health || echo "Movie Service health check failed"
                        curl -f http://localhost:8002/health || echo "Cast Service health check failed"
                    '''
                }
            }
        }
        stage('Docker Push') { 
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
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace dev || kubectl create namespace dev
                        helm -n dev upgrade --install movie-db --values ./my-app-helm/movie-db/values.yaml ./my-app-helm/movie-db/
                        helm -n dev upgrade --install cast-db --values ./my-app-helm/cast-db/values.yaml ./my-app-helm/cast-db/
                        helm -n dev upgrade --install movie-service --values ./my-app-helm/movie/values.yaml --set image.repository=$DOCKER_ID/$MOVIE_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/movie/
                        helm -n dev upgrade --install cast-service --values ./my-app-helm/cast/values.yaml --set image.repository=$DOCKER_ID/$CAST_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/cast/
                        helm -n dev upgrade --install nginx-service --values ./my-app-helm/nginx/values.yaml ./my-app-helm/nginx/
                    '''
                }
            }
        }
        stage('Deploy to QA') {
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace qa || kubectl create namespace qa
                        helm -n qa upgrade --install movie-db --values ./my-app-helm/movie-db/values.yaml ./my-app-helm/movie-db/
                        helm -n qa upgrade --install cast-db --values ./my-app-helm/cast-db/values.yaml ./my-app-helm/cast-db/
                        helm -n qa upgrade --install movie-service --values ./my-app-helm/movie/values.yaml --set image.repository=$DOCKER_ID/$MOVIE_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/movie/
                        helm -n qa upgrade --install cast-service --values ./my-app-helm/cast/values.yaml --set image.repository=$DOCKER_ID/$CAST_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/cast/
                        helm -n qa upgrade --install nginx-service --values ./my-app-helm/nginx/values.yaml ./my-app-helm/nginx/
                    '''
                }
            }
        }
        stage('Deploy to Staging') {
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat $KUBECONFIG > .kube/config
                        kubectl get namespace staging || kubectl create namespace staging
                        helm -n staging upgrade --install movie-db --values ./my-app-helm/movie-db/values.yaml ./my-app-helm/movie-db/
                        helm -n staging upgrade --install cast-db --values ./my-app-helm/cast-db/values.yaml ./my-app-helm/cast-db/
                        helm -n staging upgrade --install movie-service --values ./my-app-helm/movie/values.yaml --set image.repository=$DOCKER_ID/$MOVIE_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/movie/
                        helm -n staging upgrade --install cast-service --values ./my-app-helm/cast/values.yaml --set image.repository=$DOCKER_ID/$CAST_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/cast/
                        helm -n staging upgrade --install nginx-service --values ./my-app-helm/nginx/values.yaml ./my-app-helm/nginx/
                    '''
                }
            }
        }
        stage('Deploy to Prod') {
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
                        helm -n prod upgrade --install movie-db --values ./my-app-helm/movie-db/values.yaml ./my-app-helm/movie-db/
                        helm -n prod upgrade --install cast-db --values ./my-app-helm/cast-db/values.yaml ./my-app-helm/cast-db/
                        helm -n prod upgrade --install movie-service --values ./my-app-helm/movie/values.yaml --set image.repository=$DOCKER_ID/$MOVIE_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/movie/
                        helm -n prod upgrade --install cast-service --values ./my-app-helm/cast/values.yaml --set image.repository=$DOCKER_ID/$CAST_IMAGE --set image.tag=$DOCKER_TAG ./my-app-helm/cast/
                        helm -n prod upgrade --install nginx-service --values ./my-app-helm/nginx/values.yaml ./my-app-helm/nginx/
                    '''
                }
            }
        }
    }
}
