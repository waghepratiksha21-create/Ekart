pipeline {
    agent any

    environment {
        SCANNER_HOME   = tool 'sonar-scanner'
        NVD_API_KEY    = credentials('nvd-api-key')
        DOCKERHUB_CRED = credentials('dockerhub-pwd')
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Compile') {
            steps {
                sh "mvn clean compile -DskipTests=true"
            }
        }

        stage('Unit Tests') {
            steps {
                sh "mvn test"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    withSonarQubeEnv('sonar-server') {
                        sh "${SCANNER_HOME}/bin/sonar-scanner"
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                sh """
                dependency-check.sh \
                --project 'shopping-cart' \
                --scan ./ \
                --enableExperimental \
                --nvdApiKey ${NVD_API_KEY}
                """
            }
        }

        stage('Build Package') {
            steps {
                sh "mvn clean package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                sh "mvn deploy -DskipTests=true"
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    dockerImage = docker.build("myorg/shopping-cart:${env.BUILD_NUMBER}")
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-pwd') {
                        dockerImage.push()
                    }
                }
            }
        }

        stage('Configure EKS') {
            steps {
                sh "aws eks update-kubeconfig --name my-cluster --region ap-south-1"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f k8s/deployment.yaml"
            }
        }

        stage('Create LoadBalancer for Deployment') {
            steps {
                sh "kubectl apply -f k8s/service-lb.yaml"
            }
        }
    }

    post {
        success {
            echo "Pipeline SUCCESS!"
        }
        failure {
            echo "Pipeline FAILED!"
        }
        always {
            cleanWs()
        }
    }
}
