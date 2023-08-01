pipeline {

    environment { // Declaration of environment variables
    DOCKER_ID = "ardhmd" // replace this with your docker-id
    DOCKER_IMAGE = "movie-service"
    DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
    }

    agent any // Jenkins will be able to select all available agents

    stages {
        stage(' Docker Build -> Movie Service'){ // docker build image stage
            steps {
                script {
                sh '''
                docker rm -f movie_db
                docker rm -f movie_service
                cd movie-service/
                docker build -t $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG .
                cd - >/dev/null
                sleep 6
                '''
                }
            }
        }

        stage(' Docker run -> Movie DB'){ // run container from our builded image
            environment
            {
                MOVIE_DB_PASSWORD = credentials("MOVIE_DB_PASSWORD") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
                MOVIE_DB_USERNAME = credentials("MOVIE_DB_USERNAME") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }
            steps {
                script {
                sh '''
                docker run -d --name movie_db --network monReseau -e POSTGRES_USER=$MOVIE_DB_USERNAME -e POSTGRES_PASSWORD=$MOVIE_DB_PASSWORD -e POSTGRES_DB=movie_db_dev postgres:12.1-alpine
                sleep 10
                '''
                }
            }
        }

        stage(' Docker run -> Movie Service'){ // run container from our builded image
            environment
            {
                MOVIE_DB_PASSWORD = credentials("MOVIE_DB_PASSWORD") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
                MOVIE_DB_USERNAME = credentials("MOVIE_DB_USERNAME") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }
            steps {
                script {
                sh '''
                docker run -d -p 8086:8000 -e DATABASE_URI=postgresql://$MOVIE_DB_USERNAME:$MOVIE_DB_PASSWORD@movie_db/movie_db_dev --name movie_service --network monReseau $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                sleep 10
                '''
                }
            }
        }

        stage('Test Acceptance -> Movie Service'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl -I -s localhost:8086/api/v1/movies/docs | grep HTTP/1.1 | awk '{print $2}'
                    '''
                    }
            }

        }

        stage('Docker Push -> Movie Service'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$DOCKER_IMAGE:$DOCKER_TAG
                '''
                }
            }

        }

        stage('Deploiement en dev -> Movie Service'){
            environment
            {
            KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp movie-chart/values.yaml values.yaml
                    cat values.yaml
                    sed -i "0,/tag.*/s//tag: ${DOCKER_TAG}/" values.yaml
                    helm upgrade --install movie-chart-release ./movie-chart --values=values.yaml --set service.port=8002 --set db.name=movie_db_dev --namespace dev
                    '''
                    }
                }

        }

        stage('Deploiement en qa -> Movie Service'){
            environment
            {
            KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp movie-chart/values.yaml values.yaml
                    cat values.yaml
                    sed -i "0,/tag.*/s//tag: ${DOCKER_TAG}/" values.yaml
                    helm upgrade --install movie-chart-release ./movie-chart --values=values.yaml --set service.port=8004 --set db.name=movie_db_qa --namespace qa
                    '''
                    }
                }

        }

        stage('Deploiement en staging -> Movie Service'){
            environment
            {
            KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp movie-chart/values.yaml values.yaml
                    cat values.yaml
                    sed -i "0,/tag.*/s//tag: ${DOCKER_TAG}/" values.yaml
                    helm upgrade --install movie-chart-release ./movie-chart --values=values.yaml --set service.port=8006 --set db.name=movie_db_staging --namespace staging
                    '''
                    }
                }

        }

        stage('Deploiement en prod -> Movie Service'){
            environment
            {
            KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
                steps {
                // Create an Approval Button with a timeout of 15minutes.
                // this require a manuel validation in order to deploy on production environment
                        timeout(time: 15, unit: "MINUTES") {
                            input message: 'Do you want to deploy in production ?', ok: 'Yes'
                        }

                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    cp movie-chart/values.yaml values.yaml
                    cat values.yaml
                    sed -i "0,/tag.*/s//tag: ${DOCKER_TAG}/" values.yaml
                    helm upgrade --install movie-chart-release ./movie-chart --values=values.yaml --set service.port=8008 --set db.name=movie_db_prod --namespace prod
                    '''
                    }
                }
        }
    }
}
