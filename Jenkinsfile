pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'docker.io'
        DOCKER_REPO = 'denisber1984/mypolybot-app'
        DOCKER_CREDENTIALS_ID = 'dockerhub'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.build("${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort}", "-f polybot/Dockerfile polybot").inside {
                        sh 'echo Docker image built successfully'
                    }
                }
            }
        }

        stage('Parallel Stages') {
            parallel {
                stage('Unittest') {
                    steps {
                        script {
                            def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            docker.image("${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort}").inside {
                                sh 'python3 -m pytest --junitxml=${WORKSPACE}/results.xml tests/test.py'
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                junit allowEmptyResults: true, testResults: '**/results.xml'
                            }
                        }
                    }
                }

                stage('Static code linting') {
                    steps {
                        script {
                            def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                            docker.image("${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort}").inside {
                                sh 'python3 -m pylint -f parseable --reports=no polybot/*.py > pylint.log'
                            }
                        }
                    }
                    post {
                        always {
                            script {
                                sh 'cat pylint.log'
                                recordIssues(
                                    enabledForFailure: true,
                                    aggregatingResults: true,
                                    tools: [pyLint(name: 'Pylint', pattern: '**/pylint.log')]
                                )
                            }
                        }
                    }
                }
            }
        }

        stage('Snyk Security Scan') {
            options {
                timeout(time: 10, unit: 'MINUTES')
            }
            steps {
                script {
                    withCredentials([string(credentialsId: 'snyk-api-token', variable: 'SNYK_TOKEN')]) {
                        def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        sh """
                            docker run --rm -m 500m --cpus 0.5 -e SNYK_TOKEN=${SNYK_TOKEN} \
                            ${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort} \
                            bash -c 'snyk container test ${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort} \
                            --severity-threshold=high --file=polybot/Dockerfile --exclude-base-image-vulns --policy-path=./snyk-ignore.json'
                        """
                    }
                }
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                script {
                    def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    docker.withRegistry("https://${DOCKER_REGISTRY}", "${DOCKER_CREDENTIALS_ID}") {
                        docker.image("${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort}").push()
                        docker.image("${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort}").push('latest')
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sshagent(['ec2-ssh-credentials']) {
                        def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@ec2-18-156-71-145.eu-central-1.compute.amazonaws.com '
                                docker pull ${DOCKER_REPO}:latest
                                docker stop mypolybot-app || true
                                docker rm mypolybot-app || true
                                docker run -d --name mypolybot-app -p 80:80 ${DOCKER_REPO}:latest
                            '
                        """
                    }
                }
            }
        }
    }

    options {
        buildDiscarder(logRotator(daysToKeepStr: '30'))
        disableConcurrentBuilds()
        timestamps()
    }

    post {
        always {
            script {
                def gitCommitShort = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                // Clean up the built Docker images from the disk
                sh """
                    docker rmi ${DOCKER_REPO}:${env.BUILD_NUMBER}-${gitCommitShort} || true
                    docker rmi ${DOCKER_REPO}:latest || true
                """
            }
            // Clean the workspace
            cleanWs()
        }
        success {
            script {
                // Check if the results.xml file exists before trying to access it
                if (fileExists("${WORKSPACE}/results.xml")) {
                    junit allowEmptyResults: true, testResults: '**/results.xml'
                } else {
                    echo "No results.xml file found, skipping JUnit reporting."
                }
            }
        }
    }
}
