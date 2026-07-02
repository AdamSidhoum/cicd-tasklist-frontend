pipeline {
    agent any

    tools {
        nodejs 'NodeJS'
    }

    environment {
        DOCKER_IMAGE = 'adamsidhoum/cicd-tasklist-frontend'
        DOCKER_TAG = "${BUILD_NUMBER}"
        SONAR_SCANNER = tool 'SonarScanner'
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
            }
        }

        stage('Run unit tests') {
            steps {
                sh 'npm run test:coverage'
            }
            post {
                always {
                    junit 'reports/junit.xml'
                }
            }
        }

        stage('SonarQube analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh "${SONAR_SCANNER}/bin/sonar-scanner"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Build Docker image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -t ${DOCKER_IMAGE}:latest ."
            }
        }

        stage('Trivy scan') {
            steps {
                sh "trivy image --format table --exit-code 0 --severity HIGH,CRITICAL ${DOCKER_IMAGE}:${DOCKER_TAG}"
                sh "trivy image --format json -o trivy-report.json ${DOCKER_IMAGE}:${DOCKER_TAG}"
            }
        }

        stage('Generate SBOM') {
            steps {
                sh "trivy image --format spdx-json -o sbom-spdx.json ${DOCKER_IMAGE}:${DOCKER_TAG}"
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                    sh 'docker logout'
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.json,sbom-spdx.json', allowEmptyArchive: true
            sh "docker rmi ${DOCKER_IMAGE}:${DOCKER_TAG} || true"
        }
    }
}
