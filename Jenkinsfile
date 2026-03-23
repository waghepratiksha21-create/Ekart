pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'             // SonarQube scanner installation name
        NVD_API_KEY = credentials('nvd-api-key')       // Secret text for OWASP Dependency Check
    }

    tools {
        maven 'maven3'   // Must match Maven installation in Jenkins Global Tool Configuration
        jdk 'jdk17'      // Must match JDK installation name in Jenkins
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
                // Must match the SonarQube server name in Jenkins Global Configuration
                withSonarQubeEnv('sonar-server') {
                    sh """${env.SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=target/classes"""
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: "--nvdApiKey=$NVD_API_KEY",
                                odcInstallation: 'DC'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests=true'
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

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-pwd', 
                    usernameVariable: 'DOCKER_USER', 
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
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
    }

    post {
        always {
            echo 'Pipeline finished!'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
