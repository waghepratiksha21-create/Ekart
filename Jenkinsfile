pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-api-key')
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-pwd')
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile -DskipTests=true'
            }
        }

        stage('Tests & Analysis') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        script {
                            // Run tests but don't stop the pipeline on failure
                            try {
                                sh 'mvn test'
                            } catch (Exception e) {
                                echo "Unit tests failed, but pipeline continues."
                            }
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        withSonarQubeEnv('sonar-server') {
                            sh "${SCANNER_HOME}/bin/sonar-scanner"
                        }
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        sh "mvn org.owasp:dependency-check-maven:check -Dnvd.apiKey=${NVD_API_KEY}"
                    }
                }
            }
        }

        stage('Build Package') {
            steps {
                sh 'mvn clean package -DskipTests=true'
            }
        }

        stage('Deploy to Nexus') {
            steps {
                sh '/var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/maven3/bin/mvn deploy -DskipTests=true'
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh "docker build -t myapp:${env.BUILD_NUMBER} ."
                    sh "docker tag myapp:${env.BUILD_NUMBER} mydockerhubuser/myapp:${env.BUILD_NUMBER}"
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-pwd') {
                        sh "docker push mydockerhubuser/myapp:${env.BUILD_NUMBER}"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f k8s/deployment.yaml"
            }
        }

        stage('Create LoadBalancer for Deployment') {
            steps {
                sh "kubectl apply -f k8s/service.yaml"
            }
        }
    }

    post {
        always {
            echo "Pipeline finished!"
            cleanWs()
        }
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline FAILED. Check logs!"
        }
    }
}
