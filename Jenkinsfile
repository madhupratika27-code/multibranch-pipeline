pipeline {
    agent any
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
                script {
                    echo "Building branch: ${env.BRANCH_NAME}"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=python-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=$SONAR_HOST_URL
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh """
                            docker build -t madhupratika/masterimage . 
                            docker tag madhupratika/masterimage:latest madhupratika/masterimage:${BUILD_NUMBER}
                        """
                    } else if (env.BRANCH_NAME == "developer") {
                        sh """
                            docker build -t madhupratika/devimage . 
                            docker tag madhupratika/devimage:latest madhupratika/devimage:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh "trivy image --exit-code 0 --severity HIGH,CRITICAL madhupratika/masterimage:latest"
                    } else if (env.BRANCH_NAME == "developer") {
                        sh "trivy image --exit-code 0 --severity HIGH,CRITICAL madhupratika/devimage:latest"
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'

                    script {
                        if (env.BRANCH_NAME == "master") {
                            sh """
                                docker push madhupratika/masterimage:latest
                                docker push madhupratika/masterimage:${BUILD_NUMBER}
                            """
                        } else if (env.BRANCH_NAME == "developer") {
                            sh """
                                docker push madhupratika/devimage:latest
                                docker push madhupratika/devimage:${BUILD_NUMBER}
                            """
                        }
                    }
                }
            }
        }

        stage('Deploy on Docker') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh "docker rm -f masterapp || true"
                        sh "docker run -itd --name masterapp -p 8010:80 madhupratika/masterimage:latest"
                    } else if (env.BRANCH_NAME == "developer") {
                        sh "docker rm -f devapp || true"
                        sh "docker run -itd --name devapp -p 8020:80 madhupratika/devimage:latest"
                    }
                }
            }
        }

        stage('Cleanup Old Images') {
            steps {
                script {
                    if (env.BRANCH_NAME == "master") {
                        sh """
                            docker images "madhupratika/masterimage" --format "{{.Repository}}:{{.Tag}}" \
                            | grep -v "latest" \
                            | grep -v "${BUILD_NUMBER}" \
                            | xargs -r docker rmi -f
                        """
                    } else if (env.BRANCH_NAME == "developer") {
                        sh """
                            docker images "madhupratika/devimage" --format "{{.Repository}}:{{.Tag}}" \
                            | grep -v "latest" \
                            | grep -v "${BUILD_NUMBER}" \
                            | xargs -r docker rmi -f
                        """
                    }

                    // Clean dangling layers also
                    sh "docker image prune -f"
                }
            }
        }
    }
}
