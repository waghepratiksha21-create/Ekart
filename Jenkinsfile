pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'               // Sonar Scanner tool name in Jenkins
        NVD_API_KEY = credentials('nvd-api-key')         // Secret text for OWASP Dependency Check
    }

    tools {
        maven 'maven3'   // Maven installation name in Jenkins
        jdk 'jdk17'      // JDK installation name in Jenkins
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
                withSonarQubeEnv('sonar-server') {   // Name of SonarQube installation in Jenkins
                    sh """
                        ${env.SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=target/classes
                    """
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

        stage('Build') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Deploy to Nexus') {
            steps {
                // Username/Password credentials required
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', 
                                                  usernameVariable: 'NEXUS_USER', 
                                                  passwordVariable: 'NEXUS_PASSWORD')]) {
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

        stage('Build and Tag Docker Image') {
            steps {
                sh 'docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile .'
            }
        }

        stage('Push Docker Image to Hub') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'DOCKERHUB_PASSWORD')]) {
                    sh 'docker login -u waghepratiksha21 -p ${DOCKERHUB_PASSWORD}'
                    sh 'docker push waghepratiksha21/ekart:latest'
                }
            }
        }

        stage('EKS and Kubectl Configuration') {
            steps {
                sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }

    } // stages

    post {
        always {
            echo 'Pipeline finished'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }

}
