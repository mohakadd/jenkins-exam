pipeline {
    agent any

    environment {
        // Variables pour les images Docker
        CAST_IMAGE = "mohakadd/cast-service"
        MOVIE_IMAGE = "mohakadd/movie-service"
        DOCKER_TAG = "latest"
    }

    stages {
        stage('Checkout') {
            steps {
                // Récupère le code source depuis le dépôt
                checkout scm
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build Cast Service') {
                    steps {
                        script {
                            // Construction de l'image Docker pour cast-service
                            docker.build("${CAST_IMAGE}:${DOCKER_TAG}", "cast-service")
                        }
                    }
                }

                stage('Build Movie Service') {
                    steps {
                        script {
                            // Construction de l'image Docker pour movie-service
                            docker.build("${MOVIE_IMAGE}:${DOCKER_TAG}", "movie-service")
                        }
                    }
                }
            }
        }

        stage('Push Docker Images') {
            steps {
                script {
                    // Récupère les informations d'identification Docker Hub depuis Jenkins
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', 
                                                     usernameVariable: 'DOCKER_ID', 
                                                     passwordVariable: 'DOCKER_PASS')]) {
                        // Login Docker
                        sh """
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_ID" --password-stdin
                        """
                        
                        // Push cast-service
                        sh """
                            docker push ${CAST_IMAGE}:${DOCKER_TAG}
                        """
                        
                        // Push movie-service
                        sh """
                            docker push ${MOVIE_IMAGE}:${DOCKER_TAG}
                        """
                        
                        // Logout Docker
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