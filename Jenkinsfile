pipeline {
    agent any

    tools {
        nodejs 'NodeJS20'
    }

    environment {
        APP_NAME        = 'curso-devops-lab3'
        DOCKERHUB_USER  = 'kaptor73'
        GITHUB_USER     = 'priscilagonzalez-blip'
        GITHUB_REGISTRY = 'ghcr.io'
        DOCKER_IMAGE_DH = "${DOCKERHUB_USER}/${APP_NAME}"
        DOCKER_IMAGE_GH = "${GITHUB_REGISTRY}/${GITHUB_USER}/${APP_NAME}"
        SEMANTIC_VERSION = "1.0.${BUILD_NUMBER}"
        NAMESPACE       = 'pgonzalez'
        SONAR_PROJECT   = 'curso-devops-lab3'
    }

    stages {

        stage('Instalación de dependencias') {
            steps {
                sh 'npm install'
            }
        }

        stage('Ejecución de pruebas') {
            steps {
                sh 'npm run test:cov'
            }
            post {
                always {
                    junit 'coverage/junit.xml'
                }
            }
        }

        stage('Análisis SonarQube') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        npx sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT} \
                          -Dsonar.sources=src \
                          -Dsonar.tests=src \
                          -Dsonar.test.inclusions=**/*.spec.ts \
                          -Dsonar.typescript.lcov.reportPaths=coverage/lcov.info \
                          -Dsonar.coverage.exclusions=**/*.spec.ts,**/main.ts
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build de la aplicación') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Construcción imagen Docker') {
            steps {
                sh "docker build -t ${APP_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Push a DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh """
                        echo "${DH_PASS}" | docker login -u "${DH_USER}" --password-stdin

                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_DH}:latest
                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_DH}:${SEMANTIC_VERSION}
                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_DH}:${BUILD_NUMBER}

                        docker push ${DOCKER_IMAGE_DH}:latest
                        docker push ${DOCKER_IMAGE_DH}:${SEMANTIC_VERSION}
                        docker push ${DOCKER_IMAGE_DH}:${BUILD_NUMBER}

                        docker logout
                    """
                }
            }
        }

        stage('Push a GitHub Packages') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'github-credentials',
                    usernameVariable: 'GH_USER',
                    passwordVariable: 'GH_TOKEN'
                )]) {
                    sh """
                        echo "${GH_TOKEN}" | docker login ${GITHUB_REGISTRY} -u "${GH_USER}" --password-stdin

                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_GH}:latest
                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_GH}:${SEMANTIC_VERSION}
                        docker tag ${APP_NAME}:${BUILD_NUMBER} ${DOCKER_IMAGE_GH}:${BUILD_NUMBER}

                        docker push ${DOCKER_IMAGE_GH}:latest
                        docker push ${DOCKER_IMAGE_GH}:${SEMANTIC_VERSION}
                        docker push ${DOCKER_IMAGE_GH}:${BUILD_NUMBER}

                        docker logout ${GITHUB_REGISTRY}
                    """
                }
            }
        }

        stage('Deploy en Kubernetes') {
            steps {
                sh """
                    kubectl set image deployment/${APP_NAME} \
                        ${APP_NAME}=${DOCKER_IMAGE_GH}:${BUILD_NUMBER} \
                        -n ${NAMESPACE}

                    kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE}
                """
            }
        }
    }

    post {
        success {
            echo "Pipeline OK — Build: ${BUILD_NUMBER} — Version: ${SEMANTIC_VERSION}"
        }
        failure {
            echo "Pipeline fallido en build: ${BUILD_NUMBER}"
        }
        always {
            sh 'docker image prune -f'
        }
    }
}