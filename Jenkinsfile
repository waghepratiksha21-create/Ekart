pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    tools {
        maven 'maven3'
        jdk 'jdk-17'
    }

    stages {
        stage('Checkout') {
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
                sh "mvn test"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh "${env.SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=target/classes"
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                // Make sure 'DC' is configured in Jenkins -> Global Tool Configuration
                dependencyCheck additionalArguments: '--format HTML', odcInstallation: 'DC', skipOnError: true
            }
        }

        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk-17', maven: 'maven3') {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'DOCKER_HUB_PWD')]) {
                        sh "docker login -u waghepratiksha21 -p ${DOCKER_HUB_PWD}"
                        sh "docker push waghepratiksha21/ekart:latest"
                    }
                }
            }
        }

        stage('Configure EKS') {
            steps {
                script {
                    sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh 'kubectl apply -f deploymentservice.yml'
                }
            }
        }
    }

    post {
        always {
            node {
                echo "Cleaning workspace..."
                cleanWs()
            }
        }
        failure {
            echo "Pipeline failed!"
        }
    }
}
