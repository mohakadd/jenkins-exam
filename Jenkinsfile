pipeline {
    agent any

    environment {
        // Variables pour les images Docker
        DOCKER_ID = "mohakadd"
        CAST_IMAGE = "mohakadd/cast-service"
        MOVIE_IMAGE = "mohakadd/movie-service"
        DOCKER_TAG = "latest"
        // Variable pour la branche courrente
        BRANCH_NAME = "${GIT_BRANCH.split("/")[1]}"
        // Environements
        IP_PROD = "52.213.203.192"
        IP_DEV = "52.213.203.192"
        IP_QA = "52.213.203.192"
        IP_STAGING = "52.213.203.192"
        NODEPORT_PROD = "30000"
        NODEPORT_DEV = "30001"
        NODEPORT_QA = "30002"
        NODEPORT_STAGING = "30003"
    }

    stages {
        stage('Checkout'){
            steps {
                checkout scm
                script{
                    sh"""
                     echo "Branche du projet : ${BRANCH_NAME}"
                    """
                }
            }
        }

        stage('Build Docker Images'){
            parallel {
                stage('Build Cast Service'){
                    steps {
                        dir('cast-service'){
                            script {
                                sh """
                                    docker build -t ${CAST_IMAGE}:${DOCKER_TAG} .
                                """
                            }
                        }
                    }
                }

                stage('Build Movie Service'){
                    steps {
                        dir('movie-service'){
                            script {
                                sh """
                                    docker build -t ${MOVIE_IMAGE}:${DOCKER_TAG} .
                                """
                            }
                        }
                    }
                }
            }
        }


        stage('Run Build for tests'){
            steps {
                script{
                    sh """
                        docker compose -p movie-test -f docker-compose.yml up -d
                        sleep 30
                    """
                }
            }
        }

        stage('Acceptance Tests'){
            parallel{
                stage('Acceptance Test MOVIE SERVICE'){
                    steps {
                        script{
                            def response = sh(script: "curl -s http://localhost:8081/api/v1/movies/docs | grep 'FastAPI - Swagger UI'", returnStatus: true)
                            if (response != 0) {
                                error("Acceptance Test MOVIE SERVICE failed: 'FastAPI - Swagger UI' not found in response")
                            }
                        }
                    }
                }

                stage('Acceptance Test CAST SERVICE'){
                    steps {
                        script{
                            def response = sh(script: "curl -s http://localhost:8081/api/v1/casts/docs | grep 'FastAPI - Swagger UI'", returnStatus: true)
                            if (response != 0) {
                                error("Acceptance Test CAST SERVICE failed: 'FastAPI - Swagger UI' not found in response")
                            }
                        }
                    }
                }
            }
        }

        stage('Down Build test'){
            steps {
                script{
                    sh """
                        docker compose -p movie-test -f docker-compose.yml down
                        sleep 10
                    """
                }
            }
        }

        stage('Push Images'){
            parallel{
                stage('Push CAST Image'){
                    environment {
                        DOCKER_PASS = credentials("DOCKER_HUB_PASS") // Récupère le mot de passe Docker depuis les credentials Jenkins
                    }
                    steps {
                        script {
                            // Connexion à Docker Hub
                            sh """
                                docker login -u ${DOCKER_ID} -p ${DOCKER_PASS}
                            """

                            // Pousser cast-service
                            sh """
                                docker push ${CAST_IMAGE}:${DOCKER_TAG}
                            """

                            // Déconnexion de Docker Hub
                            sh 'docker logout'
                        }
                    }
                }
            

                stage('Push MOVIE Image'){
                    environment {
                        DOCKER_PASS = credentials("DOCKER_HUB_PASS") // Récupère le mot de passe Docker depuis les credentials Jenkins
                    }
                    steps {
                        script {
                            // Connexion à Docker Hub
                            sh """
                                docker login -u ${DOCKER_ID} -p ${DOCKER_PASS}
                            """

                            // Pousser movie-service
                            sh """
                                docker push ${MOVIE_IMAGE}:${DOCKER_TAG}
                            """

                            // Déconnexion de Docker Hub
                            sh 'docker logout'
                        }
                    }
                }
            }
        }

        stage('Deploiement environments'){
            parallel{
                stage('Deploiement en DEV'){
                    environment {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            helm upgrade --install movie-app ./movie-app --values=values.yml --namespace dev --set nginx.nodePort=$NODEPORT_DEV
                            '''
                            echo 'Evironement de DEV: $IP_DEV:$NODEPORT_DEV'
                        }
                    }
                }

                stage('Deploiement en QA'){
                    environment {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            helm upgrade --install movie-app ./movie-app --values=values.yml --namespace qa --set nginx.nodePort=$NODEPORT_QA
                            '''
                            echo 'Evironement de QA: $IP_QA:$NODEPORT_QA'
                        }
                    }
                }
            
                stage('Deploiement en STAGING'){
                    environment {
                        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
                    }
                    steps {
                        script {
                            sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            helm upgrade --install movie-app ./movie-app --values=values.yml --namespace staging --set nginx.nodePort=$NODEPORT_STAGING
                            '''
                            echo 'Evironement de STAGING: $IP_STAGING:$NODEPORT_STAGING'
                        }
                    }
                }
            }
        }

        stage('Deploiement en PROD'){
            when {
                expression { env.BRANCH_NAME == 'master' }
            }
            environment {
                KUBECONFIG = credentials("config")
            }
            steps {
                script {
                    try {
                        timeout(time: 15, unit: "MINUTES") {
                            input message: 'Do you want to deploy in production ?', ok: 'Yes'
                        }
                    } catch (Exception e) {
                        echo "Aucune validation reçue dans le délai imparti. Le déploiement en PROD est annulé, mais le pipeline continue."
                        return // Sortie anticipée de l'étape sans échec du pipeline
                    }

                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    ls
                    cat $KUBECONFIG > .kube/config
                    helm upgrade --install movie-app ./movie-app --values=values.yml --namespace prod --set nginx.nodePort=$NODEPORT_PROD
                    '''
                    echo 'Evironement de PROD: $IP_PROD:$NODEPORT_PROD'
                }
            }
        }

    }

    post {
        success {
            echo 'Les environnements sont déployés'
        
            script {
                currentBuild.description = """
                    <ul>
                        <li><a href='http://${IP_PROD}:${NODEPORT_PROD}/api/v1/movies/docs' target='_blank'>🔗 Accéder à l'application en PROD</a></li>
                        <li><a href='http://${IP_DEV}:${NODEPORT_DEV}/api/v1/movies/docs' target='_blank'>🔗 Accéder à l'application en DEV</a></li>
                        <li><a href='http://${IP_QA}:${NODEPORT_QA}/api/v1/movies/docs' target='_blank'>🔗 Accéder à l'application en QA</a></li>
                        <li><a href='http://${IP_STAGING}:${NODEPORT_STAGING}/api/v1/movies/docs' target='_blank'>🔗 Accéder à l'application en STAGING</a></li>
                    </ul>
                """
            }
        }
        failure {
            echo 'Échec du pipeline.'
        }
    }
}