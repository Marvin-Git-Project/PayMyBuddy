pipeline {

    // Variables d'environnement globales disponibles dans toutes les étapes
    environment {
        DOCKER_ID      = "marvindocker28"
        IMAGE_NAME     = "paymybuddy"
        IMAGE_TAG      = "latest"
        STAGING_HOST   = "35.180.87.90"
        PROD_HOST      = "52.47.110.5"
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

        // Étape 3 : Compilation, packaging et push sur Docker Hub
        // -------------------------------------------------------
        stage('Build & Push') {
            agent any
            environment {
                DOCKERHUB_CREDENTIALS = credentials('dockerhub')
            }
            steps {
                sh 'mvn package -DskipTests'
                sh 'docker build -t ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG} .'
                sh 'echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin'
                sh 'docker push ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG}'
            }
        }

        // Étape 4 : Déploiement en staging (branche main uniquement)
        // -------------------------------------------------------
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            agent any
            environment {
                SSH_STAGING = credentials('ssh_staging')
            }
            steps {
                sshagent(['ssh_staging']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ubuntu@${STAGING_HOST} '
                            docker pull ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG} &&
                            docker stop ${IMAGE_NAME} || true &&
                            docker rm ${IMAGE_NAME} || true &&
                            docker run -d --name ${IMAGE_NAME} -p 80:8080 ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        // Étape 5 : Validation du déploiement staging
        // -------------------------------------------------------
        stage('Validate Staging') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sh """
                    sleep 10
                    curl -f http://${STAGING_HOST} || exit 1
                """
            }
        }

        // Étape 6 : Déploiement en production (branche main uniquement)
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
                            docker run -d --name ${IMAGE_NAME} -p 80:8080 ${DOCKER_ID}/${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        // Étape 7 : Validation du déploiement production
        // -------------------------------------------------------
        stage('Validate Production') {
            when {
                branch 'main'
            }
            agent any
            steps {
                sh """
                    sleep 10
                    curl -f http://${PROD_HOST} || exit 1
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
