pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-api-key')
    }

    tools {
        maven 'maven3'
        jdk 'jdk8'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test -DskipTests=true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
                    sh '''#!/bin/bash
                    docker run --rm -v "$PWD":/usr/src \
                        -e SONAR_TOKEN="$SONAR_TOKEN" \
                        sonarsource/sonar-scanner-cli:latest \
                        sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.sources=/usr/src/com/reljicd \
                        -Dsonar.java.binaries=/usr/src/target/classes \
                        -Dsonar.host.url=http://13.233.125.170:9000 \
                        -Dsonar.token="$SONAR_TOKEN" \
                        -Dsonar.scm.provider=git
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: "--nvdApiKey=$NVD_API_KEY",
                                    odcInstallation: 'DC'
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
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk8', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh 'docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile .'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'DOCKERHUB_PWD')]) {
                        sh 'docker login -u waghepratiksha21 -p "$DOCKERHUB_PWD"'
                    }
                    sh 'docker push waghepratiksha21/ekart:latest'
                }
            }
        }

        stage('EKS Configuration') {
            steps {
                sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }

    } // closes stages
} // closes pipeline
