pipeline {
    agent any

    environment {
        NVD_API_KEY = credentials('nvd-api-key')           // OWASP NVD API key
        SONAR_TOKEN = credentials('sonarcloud-token')     // SonarCloud token stored in Jenkins
        SONAR_ORG = "<your-org-key>"                      // Replace with your SonarCloud organization key
    }

    tools {
        maven 'maven3'   // Maven installation
        jdk 'jdk8'       // Use Java 8 for Spring Boot 1.5
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh "mvn clean compile"
            }
        }

        stage('Unit Tests') {
            steps {
                sh "mvn test -DskipTests"  // Remove -DskipTests if you want to run tests
            }
        }

        stage('SonarCloud Analysis') {
            agent {
                docker {
                    image 'sonarsource/sonar-scanner-cli:latest'
                    args '-v $WORKSPACE:/usr/src'
                }
            }
            steps {
                sh """
                    sonar-scanner \
                    -Dsonar.projectKey=EKART \
                    -Dsonar.organization=${SONAR_ORG} \
                    -Dsonar.login=${SONAR_TOKEN} \
                    -Dsonar.projectName=EKART \
                    -Dsonar.sources=/usr/src \
                    -Dsonar.java.binaries=/usr/src/target/classes
                """
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: "--nvdApiKey=$NVD_API_KEY",
                                    odcInstallation: 'DC'
                }
            }
        }

        stage('Build Package') {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withMaven(
                    globalMavenSettingsConfig: 'global-maven',
                    maven: 'maven3',
                    jdk: 'jdk8'
                ) {
                    sh "mvn clean deploy -DskipTests"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile ."
            }
        }

        stage('Push Docker Image to Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-pwd', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                        docker login -u $DOCKER_USER -p $DOCKER_PASS
                        docker push waghepratiksha21/ekart:latest
                    """
                }
            }
        }

        stage('EKS Configuration') {
            steps {
                sh "aws eks update-kubeconfig --region ap-south-1 --name project-cluster"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh "kubectl apply -f deploymentservice.yml"
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed. Check logs for details."
        }
    }
}
