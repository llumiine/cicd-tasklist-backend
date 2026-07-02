pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        SONAR_TOKEN = credentials('sonarqube-token')
        SONAR_HOST_URL = 'https://sonarqube.cicd.kits.ext.educentre.fr'
        IMAGE_NAME = 'llumine/tasklist-backend'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install dependencies') {
            steps {
                sh 'npm ci'
                sh 'npx prisma generate'
            }
        }

        stage('Unit tests + coverage') {
            steps {
                sh 'npx prisma generate --schema=prisma/schema-test.prisma'
                sh 'npm run test:coverage'
            }
        }

        stage('E2E tests + coverage') {
            steps {
                sh 'npm run test:e2e:coverage'
            }
        }

        stage('SonarQube analysis') {
            steps {
                sh """
                    npx sonar-scanner \
                    -Dsonar.host.url=${SONAR_HOST_URL} \
                    -Dsonar.token=${SONAR_TOKEN}
                """
            }
        }

        stage('Build Docker image') {
            steps {
                sh "docker buildx build --tag ${IMAGE_NAME}:${BUILD_NUMBER} --tag ${IMAGE_NAME}:latest --load ."
            }
        }

        stage('Trivy security scan') {
            steps {
                sh "trivy image --timeout 15m --severity CRITICAL,HIGH --format table ${IMAGE_NAME}:latest"
            }
        }

        stage('Generate SBOM') {
            steps {
                sh "trivy image --timeout 15m --format spdx-json --output sbom-spdx.json ${IMAGE_NAME}:latest"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                sh "docker push ${IMAGE_NAME}:${BUILD_NUMBER}"
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'sbom-spdx.json', allowEmptyArchive: true
            junit 'reports/junit.xml'
        }
        success {
            echo 'Pipeline réussi !'
        }
        failure {
            echo 'Pipeline échoué — vérifier les logs.'
        }
    }
}