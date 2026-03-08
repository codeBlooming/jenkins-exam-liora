pipeline {
    environment {
        DOCKER_ID    = 'devopsdsgc'
        DOCKER_IMAGE = 'dsjenkinsexam'
        MOVIE_TAG    = "movie.v.${BUILD_ID}.0"
        CAST_TAG     = "cast.v.${BUILD_ID}.0"
    }
    agent any
    stages {

        stage('Docker Build') {
            steps {
                script {
                    sh '''
                        docker rm -f movie_service cast_service || true
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:$MOVIE_TAG ./movie-service
                        docker build -t $DOCKER_ID/$DOCKER_IMAGE:$CAST_TAG  ./cast-service
                        sleep 6
                    '''
                }
            }
        }

        stage('Docker Run') {
            steps {
                script {
                    sh '''
                        docker compose down || true
                        docker compose up -d --build
                        sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                        NGINX_IP=$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker compose ps -q nginx))
                        curl -f --retry 10 --retry-delay 5 --retry-connrefused --retry-all-errors http://${NGINX_IP}:8080/api/v1/movies/docs
                        curl -f --retry 10 --retry-delay 5 --retry-connrefused --retry-all-errors http://${NGINX_IP}:8080/api/v1/casts/docs
                    '''
                }
            }
        }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials('DOCKER_HUB_PASS')
            }
            steps {
                script {
                    sh '''
                        docker login -u $DOCKER_ID -p $DOCKER_PASS
                        docker push $DOCKER_ID/$DOCKER_IMAGE:$MOVIE_TAG
                        docker push $DOCKER_ID/$DOCKER_IMAGE:$CAST_TAG
                        docker compose down
                    '''
                }
            }
        }

        stage('Deploiement en dev') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat "$KUBECONFIG" > .kube/config

                        helm repo add bitnami https://charts.bitnami.com/bitnami --force-update

                        helm upgrade --install movie-db bitnami/postgresql --namespace dev --create-namespace \
                            --set auth.username=movie_db_username \
                            --set auth.password=movie_db_password \
                            --set auth.database=movie_db_dev

                        helm upgrade --install cast-db bitnami/postgresql --namespace dev \
                            --set auth.username=cast_db_username \
                            --set auth.password=cast_db_password \
                            --set auth.database=cast_db_dev

                        cp charts/values.yaml values-movie.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-movie.yml
                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values-movie.yml
                        cat >> values-movie.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://movie_db_username:movie_db_password@movie-db-postgresql:5432/movie_db_dev"
  - name: CAST_SERVICE_HOST_URL
    value: "http://cast-service-fastapiapp.dev.svc.cluster.local:8000/api/v1/casts/"
EOF
                        helm upgrade --install movie-service charts --values=values-movie.yml --namespace dev --set service.nodePort=30007

                        cp charts/values.yaml values-cast.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-cast.yml
                        sed -i "s+tag.*+tag: $CAST_TAG+g" values-cast.yml
                        cat >> values-cast.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://cast_db_username:cast_db_password@cast-db-postgresql:5432/cast_db_dev"
EOF
                        helm upgrade --install cast-service charts --values=values-cast.yml --namespace dev --set service.nodePort=30008
                    '''
                }
            }
        }

        stage('Deploiement en SA') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat "$KUBECONFIG" > .kube/config

                        helm repo add bitnami https://charts.bitnami.com/bitnami --force-update

                        helm upgrade --install movie-db bitnami/postgresql --namespace sa --create-namespace \
                            --set auth.username=movie_db_username \
                            --set auth.password=movie_db_password \
                            --set auth.database=movie_db_dev

                        helm upgrade --install cast-db bitnami/postgresql --namespace sa \
                            --set auth.username=cast_db_username \
                            --set auth.password=cast_db_password \
                            --set auth.database=cast_db_dev

                        cp charts/values.yaml values-movie.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-movie.yml
                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values-movie.yml
                        cat >> values-movie.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://movie_db_username:movie_db_password@movie-db-postgresql:5432/movie_db_dev"
  - name: CAST_SERVICE_HOST_URL
    value: "http://cast-service-fastapiapp.sa.svc.cluster.local:8000/api/v1/casts/"
EOF
                        helm upgrade --install movie-service charts --values=values-movie.yml --namespace sa --set service.nodePort=30009

                        cp charts/values.yaml values-cast.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-cast.yml
                        sed -i "s+tag.*+tag: $CAST_TAG+g" values-cast.yml
                        cat >> values-cast.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://cast_db_username:cast_db_password@cast-db-postgresql:5432/cast_db_dev"
EOF
                        helm upgrade --install cast-service charts --values=values-cast.yml --namespace sa --set service.nodePort=30010
                    '''
                }
            }
        }

        stage('Deploiement en staging') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat "$KUBECONFIG" > .kube/config

                        helm repo add bitnami https://charts.bitnami.com/bitnami --force-update

                        helm upgrade --install movie-db bitnami/postgresql --namespace staging --create-namespace \
                            --set auth.username=movie_db_username \
                            --set auth.password=movie_db_password \
                            --set auth.database=movie_db_dev

                        helm upgrade --install cast-db bitnami/postgresql --namespace staging \
                            --set auth.username=cast_db_username \
                            --set auth.password=cast_db_password \
                            --set auth.database=cast_db_dev

                        cp charts/values.yaml values-movie.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-movie.yml
                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values-movie.yml
                        cat >> values-movie.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://movie_db_username:movie_db_password@movie-db-postgresql:5432/movie_db_dev"
  - name: CAST_SERVICE_HOST_URL
    value: "http://cast-service-fastapiapp.staging.svc.cluster.local:8000/api/v1/casts/"
EOF
                        helm upgrade --install movie-service charts --values=values-movie.yml --namespace staging --set service.nodePort=30011

                        cp charts/values.yaml values-cast.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-cast.yml
                        sed -i "s+tag.*+tag: $CAST_TAG+g" values-cast.yml
                        cat >> values-cast.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://cast_db_username:cast_db_password@cast-db-postgresql:5432/cast_db_dev"
EOF
                        helm upgrade --install cast-service charts --values=values-cast.yml --namespace staging --set service.nodePort=30012
                    '''
                }
            }
        }

        stage('Deploiement en prod') {
            environment {
                KUBECONFIG = credentials('config')
            }
            when {
                expression { GIT_BRANCH == 'origin/master' }
            }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Déployer en production ?', ok: 'Oui, déployer'
                }
                script {
                    sh '''
                        rm -Rf .kube
                        mkdir .kube
                        cat "$KUBECONFIG" > .kube/config

                        helm repo add bitnami https://charts.bitnami.com/bitnami --force-update

                        helm upgrade --install movie-db bitnami/postgresql --namespace prod --create-namespace \
                            --set auth.username=movie_db_username \
                            --set auth.password=movie_db_password \
                            --set auth.database=movie_db_dev

                        helm upgrade --install cast-db bitnami/postgresql --namespace prod \
                            --set auth.username=cast_db_username \
                            --set auth.password=cast_db_password \
                            --set auth.database=cast_db_dev

                        cp charts/values.yaml values-movie.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-movie.yml
                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values-movie.yml
                        cat >> values-movie.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://movie_db_username:movie_db_password@movie-db-postgresql:5432/movie_db_dev"
  - name: CAST_SERVICE_HOST_URL
    value: "http://cast-service-fastapiapp.prod.svc.cluster.local:8000/api/v1/casts/"
EOF
                        helm upgrade --install movie-service charts --values=values-movie.yml --namespace prod --set service.nodePort=30013

                        cp charts/values.yaml values-cast.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values-cast.yml
                        sed -i "s+tag.*+tag: $CAST_TAG+g" values-cast.yml
                        cat >> values-cast.yml << EOF
env:
  - name: DATABASE_URI
    value: "postgresql://cast_db_username:cast_db_password@cast-db-postgresql:5432/cast_db_dev"
EOF
                        helm upgrade --install cast-service charts --values=values-cast.yml --namespace prod --set service.nodePort=30014
                    '''
                }
            }
        }

    }
    post {
        success {
            mail to: "gch4rles@gmail.com",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} a réussi",
                body: "Le pipeline s'est terminé avec succès. Consultez la console Jenkins : ${env.BUILD_URL}"
        }
        failure {
            mail to: "gch4rles@gmail.com",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} a échoué",
                body: "Pour plus d'informations, consultez la console Jenkins : ${env.BUILD_URL}"
        }
    }
}
