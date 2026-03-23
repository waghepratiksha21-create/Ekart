pipeline {
    agent any

    environment {
        // Maven and Java tools
        MAVEN_HOME = tool name: 'Maven', type: 'maven'
        JAVA_HOME  = tool name: 'JDK 21', type: 'jdk'
        PATH       = "${JAVA_HOME}/bin:${MAVEN_HOME}/bin:${env.PATH}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
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
            environment {
                SONAR_SCANNER_HOME = tool name: 'Sonar Scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
            }
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh "${SONAR_SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectKey=EKART -Dsonar.projectName=EKART -Dsonar.java.binaries=target/classes"
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                // Using secret credentials securely
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck(
                        odcInstallation: 'DC',
                        nvdApiKey: NVD_API_KEY
                    )
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
                // Use Jenkins credentials for Nexus
                withCredentials([
                    usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')
                ]) {
                    sh """
                        mvn deploy -DskipTests=true \
                        -DaltDeploymentRepository=maven-snapshots::default::http://65.2.152.55:8081/repository/maven-snapshots/ \
                        -Dnexus.username="\\\$NEXUS_USER" \
                        -Dnexus.password="\\\$NEXUS_PASSWORD"
                    """
                }
            }
        }

        stage('Build and Tag Docker Image') {
            steps {
                sh 'docker build -t ekart-app:latest .'
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh """
                        echo "\\$DOCKER_PASSWORD" | docker login -u "\\$DOCKER_USER" --password-stdin
                        docker tag ekart-app:latest dockerhub_user/ekart-app:latest
                        docker push dockerhub_user/ekart-app:latest
                    """
                }
            }
        }

        stage('EKS Configuration') {
            steps {
                sh 'aws eks update-kubeconfig --region us-east-1 --name ekart-cluster'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f k8s/deployment.yaml'
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
