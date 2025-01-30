pipeline {
    agent any

    environment {
        // Variables pour les images Docker
        DOCKER_ID = "mohakadd"
        CAST_IMAGE = "mohakadd/cast-service"
        MOVIE_IMAGE = "mohakadd/movie-service"
        DOCKER_TAG = "latest"
    }

    stages {
        stage('Checkout'){
            steps {
                checkout scm
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


        stage('Run Build test'){
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
                            sh """
                                curl http://localhost:8081/api/v1/movies/docs
                            """
                        }
                    }
                }

                stage('Acceptance Test CAST SERVICE'){
                    steps {
                        script{
                            sh """
                                curl http://localhost:8081/api/v1/casts/docs
                            """
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
                            helm upgrade --install movie-app ./movie-app --values=values.yml --namespace dev --set nginx.nodePort=30001
                            '''
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
                            helm upgrade --install movie-app ./movie-app --values=values.yml --namespace qa --set nginx.nodePort=30002
                            '''
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
                            helm upgrade --install movie-app ./movie-app --values=values.yml --namespace staging --set nginx.nodePort=30003
                            '''
                        }
                    }
                }
            }
        }

        stage('Deploiement en PROD'){
            environment {
                KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
            }
            steps {
                script {

                    // Check if the branch is master
                    def isMasterBranch = BRANCH_NAME == 'master'
        
                    // If on master, request manual approval
                    if (isMasterBranch) {
                        timeout(time: 15, unit: "MINUTES") {
                            input message: 'Do you want to deploy in production?', ok: 'Yes'
                        }
                        script {
                        sh '''
                            rm -Rf .kube
                            mkdir .kube
                            ls
                            cat $KUBECONFIG > .kube/config
                            helm upgrade --install movie-app ./movie-app --values=values.yml --namespace prod --set nginx.nodePort=30001
                        '''
                        }
                    } else {
                        echo "Not deploying to production because the build is not on the master branch, branch name:"
                        echo BRANCH_NAME
                    }
                }
            }
        }

}

    post {
        success {
            echo 'Les environements sont déployés'
        }
        failure {
            echo 'Échec du pipline.'
        }
    }
}
