pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'    // SonarQube scanner
        MAVEN_HOME   = tool 'maven3'           // Maven
        JAVA_HOME    = tool 'jdk17'            // JDK
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-pwd')
        NVD_API_KEY = credentials('nvd-api-key')
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
    }

    stages {

        stage('Checkout SCM') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn clean compile"
            }
        }

        stage('Tests & Analysis') {
            parallel {
                
                stage('Unit Tests') {
                    steps {
                        script {
                            // Run unit tests
                            try {
                                sh "${MAVEN_HOME}/bin/mvn test"
                            } catch (err) {
                                echo "Unit tests failed, marking stage failed."
                                currentBuild.result = 'FAILURE'
                                throw err
                            }
                        }
                    }
                }

                stage('SonarQube Analysis') {
                    steps {
                        withSonarQubeEnv('sonar-server') {
                            sh """
                                ${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=shopping-cart \
                                -Dsonar.projectName="Shopping Cart" \
                                -Dsonar.projectVersion=${BUILD_NUMBER} \
                                -Dsonar.sources=src/main/java \
                                -Dsonar.java.binaries=target/classes
                            """
                        }
                    }
                }

                stage('OWASP Dependency Check') {
                    steps {
                        sh "${MAVEN_HOME}/bin/mvn org.owasp:dependency-check-maven:check -Dnvd.api.key=${NVD_API_KEY}"
                    }
                }
            }
        }

        stage('Build Package') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn package -DskipTests"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn deploy -DskipTests=true"
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh "docker build -t myrepo/shopping-cart:${BUILD_NUMBER} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-pwd') {
                        sh "docker push myrepo/shopping-cart:${BUILD_NUMBER}"
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
                sh "kubectl expose deployment shopping-cart --type=LoadBalancer --name=shopping-cart-lb"
            }
        }
    }

    post {
        always {
            echo "Pipeline finished!"
            cleanWs()
        }
        failure {
            echo "Pipeline FAILED. Check logs!"
        }
    }
}
