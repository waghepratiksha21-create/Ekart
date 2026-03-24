pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-api-key')
        DOCKERHUB_USER = 'youngminds73'
        DOCKERHUB_PWD = credentials('dockerhub-pwd')
    }

    tools {
        maven 'maven3'
        jdk 'jdk-17'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/ygminds73/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh "${tool 'maven3'}/bin/mvn compile"
            }
        }

        stage('Parallel Tests & Analysis') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh "${tool 'maven3'}/bin/mvn test"
                            }
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                withSonarQubeEnv('sonar-scanner') {
                                    sh "${SCANNER_HOME}/bin/sonar-scanner " +
                                       "-Dsonar.projectKey=EKART " +
                                       "-Dsonar.projectName=EKART " +
                                       "-Dsonar.java.binaries=target/classes"
                                }
                            }
                        }
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        script {
                            catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                                sh "${tool 'maven3'}/bin/mvn org.owasp:dependency-check-maven:check -Dnvd.api.key=${NVD_API_KEY}"
                            }
                        }
                    }
                }
            }
        }

        stage('Build Package') {
            steps {
                sh "${tool 'maven3'}/bin/mvn package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                script {
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        sh "${tool 'maven3'}/bin/mvn deploy -DskipTests=true"
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        sh "docker build -t ${DOCKERHUB_USER}/ekart:latest -f docker/Dockerfile ."
                        sh "echo ${DOCKERHUB_PWD} | docker login -u ${DOCKERHUB_USER} --password-stdin"
                        sh "docker push ${DOCKERHUB_USER}/ekart:latest"
                    }
                }
            }
        }

        stage('EKS & Kubernetes Deploy') {
            steps {
                script {
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
                        sh 'kubectl apply -f deploymentservice.yml'
                    }
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Pipeline finished. Cleaning workspace."
                cleanWs()
            }
        }
    }
}
