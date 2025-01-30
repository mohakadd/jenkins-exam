pipeline {
    agent any

    environment {
        // Variables pour les images Docker
        DOCKER_ID="mohakadd"
        CAST_IMAGE = "mohakadd/cast-service"
        MOVIE_IMAGE = "mohakadd/movie-service"
        DOCKER_TAG = "latest"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Cast Service') {
                    steps {
                        dir('cast-service') {
                            script {
                                sh """
                                    docker build -t ${CAST_IMAGE}:${DOCKER_TAG} .
                                """
                            }
                        }
                    }
                }

                stage('Build Movie Service') {
                    steps {
                        dir('movie-service') {
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

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials',
                                                 usernameVariable: 'DOCKER_ID',
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        // Login à Docker Hub
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin
                        """

                        // Pousser cast-service
                        sh """
                            docker push ${CAST_IMAGE}:${DOCKER_TAG}
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

    post {
        success {
            echo 'Images Docker construites et poussées avec succès.'
        }
        failure {
            echo 'Échec de la construction ou du push des images Docker.'
        }
    }
}