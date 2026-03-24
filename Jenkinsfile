pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_USER = 'youngminds73'
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
                sh 'mvn compile'
            }
        }

        stage('Parallel Tests & Analysis') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        // This allows pipeline to continue even if tests fail
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh 'mvn test'
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            withSonarQubeEnv('sonar-scanner') {
                                sh """${SCANNER_HOME}/bin/sonar-scanner \
                                    -Dsonar.projectKey=EKART \
                                    -Dsonar.projectName=EKART \
                                    -Dsonar.java.binaries=target/classes"""
                            }
                        }
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                                sh 'mvn org.owasp:dependency-check-maven:check -Dnvd.api.key=$NVD_API_KEY'
                            }
                        }
                    }
                }
            }
        }

        stage('Build Package') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Deploy to Nexus') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh 'docker build -t youngminds73/ekart:latest -f docker/Dockerfile .'
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'DOCKERHUB_PWD')]) {
                        sh 'echo $DOCKERHUB_PWD | docker login -u youngminds73 --password-stdin'
                    }
                    sh 'docker push youngminds73/ekart:latest'
                }
            }
        }

        stage('EKS & Kubernetes Deploy') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
                    sh 'kubectl apply -f deploymentservice.yml'
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            echo "Pipeline finished"
        }
    }
}
