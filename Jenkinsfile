pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'            // SonarQube scanner installation
        NVD_API_KEY = credentials('nvd-api-key')      // OWASP NVD API key
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
                sh "mvn test -DskipTests"  // If you want to skip, otherwise remove -DskipTests
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=EKART \
                    -Dsonar.projectName=EKART \
                    -Dsonar.sources=. \
                    -Dsonar.java.binaries=target/classes
                    """
                }
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
                script {
                    sh "docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile ."
                }
            }
        }

        stage('Push Docker Image to Hub') {
            steps {
                script {
                    // Use username/password credentials stored in Jenkins
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-pwd', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    }
                    sh "docker push waghepratiksha21/ekart:latest"
                }
            }
        }

        stage('EKS Configuration') {
            steps {
                script {
                    sh "aws eks update-kubeconfig --region ap-south-1 --name project-cluster"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    sh "kubectl apply -f deploymentservice.yml"
                }
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
