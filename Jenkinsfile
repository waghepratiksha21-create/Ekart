pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        MAVEN_HOME = tool 'maven3'
        JAVA_HOME = tool 'jdk17'
        PATH = "${JAVA_HOME}/bin:${MAVEN_HOME}/bin:${env.PATH}"
        NVD_API_KEY = credentials('nvd-api-key')
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Build & Parallel Tests') {
            steps {
                script {
                    // Build first
                    sh 'mvn clean install -DskipTests=true'

                    // Run tests, Sonar, OWASP in parallel
                    parallel (
                        'Unit Tests': {
                            sh 'mvn test'
                        },
                        'SonarQube Analysis': {
                            withSonarQubeEnv('sonar-server') {
                                sh "${SCANNER_HOME}/bin/sonar-scanner"
                            }
                        },
                        'OWASP Dependency Check': {
                            sh "mvn org.owasp:dependency-check-maven:check -Dnvd.apiKey=${NVD_API_KEY}"
                        }
                    )
                }
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-cred', 
                                                  usernameVariable: 'NEXUS_USER', 
                                                  passwordVariable: 'NEXUS_PSW')]) {
                    sh """
                        mvn deploy -DskipTests=true \
                        -Dnexus.username=$NEXUS_USER \
                        -Dnexus.password=$NEXUS_PSW
                    """
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    def imageTag = "shopping-cart:${env.BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                    env.IMAGE_TAG = imageTag
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-pwd', 
                                                  usernameVariable: 'DOCKER_USER', 
                                                  passwordVariable: 'DOCKER_PSW')]) {
                    sh """
                        echo $DOCKER_PSW | docker login -u $DOCKER_USER --password-stdin
                        docker push ${env.IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh """
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                    """
                }
            }
        }

        stage('Create LoadBalancer for Deployment') {
            steps {
                script {
                    sh "kubectl expose deployment shopping-cart --type=LoadBalancer --name=shopping-cart-lb"
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Pipeline finished!"
            }
            cleanWs()
        }
        success {
            echo "Pipeline SUCCESS"
        }
        failure {
            echo "Pipeline FAILED. Check logs!"
        }
    }
}
