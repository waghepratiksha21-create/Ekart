pipeline {
    agent any

    environment {
        MAVEN_HOME = tool 'maven3'
        SCANNER_HOME = tool 'sonar-scanner'
        JDK_HOME = tool 'jdk17'
        DOCKERHUB_USER = credentials('dockerhub-user')
        DOCKERHUB_PWD = credentials('dockerhub-pwd')
        NVD_API_KEY = credentials('nvd-api-key')
        NEXUS_REPO = 'http://nexus.example.com/repository/maven-releases/'
        SONAR_PROJECT_KEY = 'shopping-cart'
        SONAR_PROJECT_NAME = 'shopping-cart'
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh "${MAVEN_HOME}/bin/mvn clean package -DskipTests=true"
                }
            }
        }

        stage('Parallel Tests & Analysis') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh "${MAVEN_HOME}/bin/mvn test"
                        }
                    }
                }
                stage('SonarQube') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            sh """${SCANNER_HOME}/bin/sonar-scanner \
                                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                                -Dsonar.sources=src \
                                -Dsonar.java.binaries=target/classes"""
                        }
                    }
                }
                stage('OWASP Dependency Check') {
                    steps {
                        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                            withEnv(["NVD_API_KEY=${NVD_API_KEY}"]) {
                                sh "${MAVEN_HOME}/bin/mvn org.owasp:dependency-check-maven:check -Dnvd.api.key=\$NVD_API_KEY"
                            }
                        }
                    }
                }
            }
        }

        stage('Deploy to Nexus') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh "${MAVEN_HOME}/bin/mvn deploy -DaltDeploymentRepository=nexus::default::${NEXUS_REPO}"
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh """
                        docker build -t ${DOCKERHUB_USER}/shopping-cart:latest .
                        docker tag ${DOCKERHUB_USER}/shopping-cart:latest ${DOCKERHUB_USER}/shopping-cart:\$(git rev-parse --short HEAD)
                        echo "${DOCKERHUB_PWD}" | docker login -u "${DOCKERHUB_USER}" --password-stdin
                        docker push ${DOCKERHUB_USER}/shopping-cart:latest
                        docker push ${DOCKERHUB_USER}/shopping-cart:\$(git rev-parse --short HEAD)
                    """
                }
            }
        }

        stage('Kubernetes Deploy') {
            steps {
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh """
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl expose deployment shopping-cart --type=LoadBalancer --name=shopping-cart-lb
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline finished. Check stage results."
            cleanWs()
        }
    }
}
