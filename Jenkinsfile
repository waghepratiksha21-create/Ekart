pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-api-key')
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }

        stage('Unit Tests') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {   // Match your SonarQube server name
                    sh "${env.SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=target/classes"
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh "mvn org.owasp:dependency-check-maven:check -DnvdApiKey=${NVD_API_KEY}"
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'nexus-creds', 
                    usernameVariable: 'NEXUS_USER', 
                    passwordVariable: 'NEXUS_PASSWORD'
                )]) {
                    sh """
                        mvn deploy \
                        -DskipTests=true \
                        -DaltDeploymentRepository=maven-snapshots::default::http://65.2.152.55:8081/repository/maven-snapshots/ \
                        -Dnexus.username=$NEXUS_USER \
                        -Dnexus.password=$NEXUS_PASSWORD
                    """
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                sh "docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile ."
            }
        }

        stage('Push Docker Image to Hub') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                    sh "docker login -u waghepratiksha21 -p ${dockerhubpwd}"
                    sh "docker push waghepratiksha21/ekart:latest"
                }
            }
        }

        stage('EKS & Kubectl Config') {
            steps {
                sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
            }
        }

        stage('Deploy to k8s') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }
    }
}
