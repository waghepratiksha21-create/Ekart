pipeline {
    agent any

    environment {
        // Tools
        MAVEN_HOME = tool 'maven3'          // Make sure Jenkins has Maven configured
        JDK_HOME = tool 'jdk17'             // Jenkins JDK
        DOCKER_IMAGE = "ekart-app"
        DOCKER_TAG = "latest"

        // SonarQube
        SONAR_HOST = "http://13.233.125.170:9000"
        SONAR_TOKEN = credentials('sonarqube-token')  // Replace with your Jenkins credential ID

        // Nexus
        NEXUS_REPO_URL = "http://your-nexus-repo/repository/maven-releases/"
        NEXUS_CREDENTIALS = credentials('nexus-credentials') // Jenkins credential ID
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    options {
        skipDefaultCheckout(true)  // We'll handle checkout manually
        timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master',
                    url: 'https://github.com/waghepratiksha21-create/Ekart.git',
                    credentialsId: '' // Add if private repo
            }
        }

        stage('Compile') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn clean compile"
            }
        }

        stage('Unit Tests & Coverage') {
            steps {
                sh """
                    ${MAVEN_HOME}/bin/mvn test jacoco:report
                """
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh """
                    docker run --rm -v "\$PWD":/usr/src \
                        -e SONAR_TOKEN=${SONAR_TOKEN} \
                        sonarsource/sonar-scanner-cli:latest \
                        sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.sources=/usr/src/src/main/java \
                        -Dsonar.tests=/usr/src/src/test/java \
                        -Dsonar.java.binaries=/usr/src/target/classes \
                        -Dsonar.coverage.jacoco.xmlReportPaths=/usr/src/target/site/jacoco/jacoco.xml \
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.token=${SONAR_TOKEN} \
                        -Dsonar.scm.provider=git
                """
            }
        }

        stage('Build Package') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn package -DskipTests"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                sh """
                    ${MAVEN_HOME}/bin/mvn deploy \
                        -DskipTests \
                        -Dnexus.url=${NEXUS_REPO_URL} \
                        -Dnexus.username=${NEXUS_CREDENTIALS_USR} \
                        -Dnexus.password=${NEXUS_CREDENTIALS_PSW}
                """
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                sh """
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker tag ${DOCKER_IMAGE}:${DOCKER_TAG} your-docker-repo/${DOCKER_IMAGE}:${DOCKER_TAG}
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASS', usernameVariable: 'DOCKER_USER')]) {
                    sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                        docker push your-docker-repo/${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
                }
            }
        }

        stage('EKS Configuration') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ap-south-1 --name your-eks-cluster
                """
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                """
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs above for details.'
        }
        always {
            cleanWs()
        }
    }
}
