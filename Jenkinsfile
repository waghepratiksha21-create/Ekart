pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        MAVEN_HOME   = tool 'maven3'
        JAVA_HOME    = tool 'jdk17'
        NVD_API_KEY  = credentials('nvd-api-key')
        DOCKERHUB_CREDENTIALS = 'dockerhub-pwd' // Docker credential ID
    }

    options {
        timestamps()
        timeout(time: 60, unit: 'MINUTES')
        skipDefaultCheckout(false)
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

        stage('Parallel Tests & Analysis') {
            parallel {

                stage('Unit Tests') {
                    steps {
                        script {
                            try {
                                sh "${MAVEN_HOME}/bin/mvn test"
                            } catch (err) {
                                echo "Unit tests failed, marking build as UNSTABLE."
                                currentBuild.result = 'UNSTABLE'
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

            } // end parallel
        }

        stage('Package') {
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
                sh "docker build -t myrepo/shopping-cart:${BUILD_NUMBER} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKERHUB_CREDENTIALS) {
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

        stage('Create LoadBalancer') {
            steps {
                sh "kubectl expose deployment shopping-cart --type=LoadBalancer --name=shopping-cart-lb"
            }
        }

    } // end stages

    post {
        always {
            echo "Pipeline finished! Cleaning workspace..."
            cleanWs()
        }
        failure {
            echo "Pipeline FAILED. Check logs!"
        }
        unstable {
            echo "Pipeline finished with UNSTABLE status due to failed unit tests."
        }
    }
}
