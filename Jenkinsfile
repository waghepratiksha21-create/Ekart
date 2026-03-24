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

    options {
        skipStagesAfterUnstable()
        timestamps()
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean install -DskipTests=true'
            }
        }

        stage('Tests & Analysis') {
            parallel {

                stage('Unit Tests') {
                    steps {
                        script {
                            try {
                                sh 'mvn test'
                            } catch (Exception e) {
                                echo "Unit tests failed but pipeline will continue."
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

        stage('Deploy to Nexus') {
            steps {
                sh '/var/lib/jenkins/tools/hudson.tasks.Maven_MavenInstallation/maven3/bin/mvn deploy -DskipTests=true'
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh 'docker build -t yourdockerhubuser/shopping-cart:${BUILD_NUMBER} .'
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                        sh 'docker push yourdockerhubuser/shopping-cart:${BUILD_NUMBER}'
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
            }
        }

        stage('Create LoadBalancer for Deployment') {
            steps {
                sh 'kubectl expose deployment shopping-cart --type=LoadBalancer --name=shopping-cart-lb'
            }
        }

    }

    post {
        always {
            echo 'Pipeline finished!'
            cleanWs()
        }
    }
}
