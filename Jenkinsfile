pipeline {

    // Variables d'environnement globales disponibles dans toutes les étapes
    environment {
        DOCKER_ID      = "marvindocker28"
        IMAGE_NAME     = "paymybuddy"
        IMAGE_TAG      = "latest"
        STAGING_HOST   = "13.38.167.219"
        PROD_HOST      = "52.47.211.209"
        SONAR_PROJECT  = "Marvin-Git-Project_PayMyBuddy"
        SONAR_ORG      = "marvin-git-project"
        SLACK_CHANNEL  = "#jenkins-notifications"
    }

    // Pas d'agent global : chaque étape déclare son propre agent Docker
    agent none

    stages {

        // Étape 1 : Tests unitaires et d'intégration avec Maven
        // -------------------------------------------------------
        stage('Tests') {
            agent {
                docker {
                    image 'maven:3.9.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        // Étape 2 : Analyse qualité du code avec SonarCloud
        // -------------------------------------------------------
        stage('SonarCloud Analysis') {
            agent {
                docker {
                    image 'maven:3.9.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            environment {
                SONAR_TOKEN = credentials('sonar_token')
            }
            steps {
                sh """
                    mvn sonar:sonar \
                        -Dsonar.projectKey=${SONAR_PROJECT} \
                        -Dsonar.organization=${SONAR_ORG} \
                        -Dsonar.host.url=https://sonarcloud.io \
                        -Dsonar.token=${SONAR_TOKEN}
                """
            }
        }

        // Étape 3a : Compilation Maven dans un container Maven
        // Le JAR produit est conservé dans le workspace partagé
        // -------------------------------------------------------
        stage('Package') {
            agent {
                docker {
                    image 'maven:3.9.6-amazoncorretto-17'
                    args '-v /root/.m2:/root/.m2'
                }
            }
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        // Étape 3b : Build de l'image Docker et push sur Docker Hub
        // Les credentials Docker Hub sont injectés via Jenkins
        // -------------------------------------------------------
        stage('Docker Build & Push') {
            agent any
            environment {
                DOCKERHUB_CREDENTIALS = credentials('dockerhub')
            }
            steps {
                sh "docker build -t ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG} ."
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                sh "docker push ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        // Étape 4 : Déploiement en staging (branche main uniquement)
        // MySQL est installé directement sur la VM staging
        // -------------------------------------------------------
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sshagent(['ssh_staging']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${STAGING_HOST} '
                            docker pull ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker stop ${IMAGE_NAME} || true &&
                            docker rm ${IMAGE_NAME} || true &&
                            docker run -d --name ${IMAGE_NAME} -p 8080:8080 \
                                --network host \
                                -e SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/db_paymybuddy \
                                -e SPRING_DATASOURCE_USERNAME=root \
                                -e SPRING_DATASOURCE_PASSWORD=password \
                                ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        // Étape 5 : Validation du déploiement staging
        // Vérifie que l'application répond bien sur le port 8080
        // -------------------------------------------------------
        stage('Validate Staging') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sh """
                    sleep 20
                    curl -f http://${STAGING_HOST}:8080 || exit 1
                """
            }
        }

        // Étape 6 : Déploiement en production (branche main uniquement)
        // MySQL est installé directement sur la VM production
        // -------------------------------------------------------
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sshagent(['ssh_prod']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${PROD_HOST} '
                            docker pull ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker stop ${IMAGE_NAME} || true &&
                            docker rm ${IMAGE_NAME} || true &&
                            docker run -d --name ${IMAGE_NAME} -p 8080:8080 \
                                --network host \
                                -e SPRING_DATASOURCE_URL=jdbc:mysql://localhost:3306/db_paymybuddy \
                                -e SPRING_DATASOURCE_USERNAME=root \
                                -e SPRING_DATASOURCE_PASSWORD=password \
                                ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        // Étape 7 : Validation du déploiement production
        // Vérifie que l'application répond bien sur le port 8080
        // -------------------------------------------------------
        stage('Validate Production') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sh """
                    sleep 20
                    curl -f http://${PROD_HOST}:8080 || exit 1
                """
            }
        }
    }

    // Notification Slack envoyée en fin de pipeline
    // Le message varie selon le statut (succès ou échec)
    // -------------------------------------------------------
    post {
        success {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'good',
                message: """
                    *Pipeline réussie* — `${env.JOB_NAME}` #${env.BUILD_NUMBER}
                    Branche : `${env.GIT_BRANCH}`
                    Durée   : ${currentBuild.durationString}
                    Détails : ${env.BUILD_URL}
                """
            )
        }
        failure {
            slackSend(
                channel: "${SLACK_CHANNEL}",
                color: 'danger',
                message: """
                    *Pipeline échouée* — `${env.JOB_NAME}` #${env.BUILD_NUMBER}
                    Branche : `${env.GIT_BRANCH}`
                    Durée   : ${currentBuild.durationString}
                    Détails : ${env.BUILD_URL}
                """
            )
        }
    }
}
