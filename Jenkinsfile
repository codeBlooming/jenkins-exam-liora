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
                        docker compose up -d
                        sleep 10
                    '''
                }
            }
        }

        stage('Test Acceptance') {
            steps {
                script {
                    sh '''
                        curl -f --retry 10 --retry-delay 5 --retry-connrefused --retry-all-errors localhost:8080/api/v1/movies/docs
                        curl -f --retry 10 --retry-delay 5 --retry-connrefused --retry-all-errors localhost:8080/api/v1/casts/docs
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
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values.yml

                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values.yml
                        helm upgrade --install movie-service charts --values=values.yml --namespace dev --create-namespace

                        sed -i "s+tag.*+tag: $CAST_TAG+g" values.yml
                        helm upgrade --install cast-service charts --values=values.yml --namespace dev
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
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values.yml

                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values.yml
                        helm upgrade --install movie-service charts --values=values.yml --namespace sa --create-namespace

                        sed -i "s+tag.*+tag: $CAST_TAG+g" values.yml
                        helm upgrade --install cast-service charts --values=values.yml --namespace sa
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
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values.yml

                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values.yml
                        helm upgrade --install movie-service charts --values=values.yml --namespace staging --create-namespace

                        sed -i "s+tag.*+tag: $CAST_TAG+g" values.yml
                        helm upgrade --install cast-service charts --values=values.yml --namespace staging
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
                        cat $KUBECONFIG > .kube/config

                        cp charts/values.yaml values.yml
                        sed -i "s+repository.*+repository: $DOCKER_ID/$DOCKER_IMAGE+g" values.yml

                        sed -i "s+tag.*+tag: $MOVIE_TAG+g" values.yml
                        helm upgrade --install movie-service charts --values=values.yml --namespace prod --create-namespace

                        sed -i "s+tag.*+tag: $CAST_TAG+g" values.yml
                        helm upgrade --install cast-service charts --values=values.yml --namespace prod
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
