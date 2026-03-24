pipeline {
    agent any

    environment {
        NVD_API_KEY = credentials('nvd-api-key') // Jenkins secret text for OWASP
        SONAR_TOKEN = credentials('sonar-token') // Jenkins secret text for self-hosted SonarQube
        SONAR_HOST = "http://your-sonarqube-server:9000" // replace with your SonarQube server URL
    }

    tools {
        maven 'maven3'
        jdk 'jdk8'
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
                sh "mvn test -DskipTests=true"
            }
        }

        stage('SonarQube analysis') {
    steps {
        withCredentials([string(credentialsId: 'sonarqube-token', variable: 'SONAR_TOKEN')]) {
            sh '''
            docker run --rm -v "$PWD":/usr/src \
                -e SONAR_TOKEN=$SONAR_TOKEN \
                sonarsource/sonar-scanner-cli:latest \
                sonar-scanner \
                -Dsonar.projectKey=EKART \
                -Dsonar.projectName=EKART \
                -Dsonar.sources=/usr/src/com/reljicd \
                -Dsonar.java.binaries=/usr/src/target/classes \
                -Dsonar.host.url=http://13.233.125.170:9000 \
                -Dsonar.token=$SONAR_TOKEN \
                -Dsonar.scm.provider=git
            '''
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
                sh "mvn clean package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk8', maven: 'maven3') {
                    sh "mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                sh "docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile ."
            }
        }

        stage('Push Docker Image to Hub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-pwd', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh "docker login -u $DOCKER_USER -p $DOCKER_PASS"
                    sh "docker push waghepratiksha21/ekart:latest"
                }
            }
        }

        stage('EKS & Kubectl Configuration') {
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
