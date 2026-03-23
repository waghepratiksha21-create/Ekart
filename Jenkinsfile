pipeline {
    agent any  // Runs on any available Jenkins agent

    environment {
        // Replace these tool names with what you have installed in Jenkins
        JAVA_HOME = tool name: 'JDK 21', type: 'jdk'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        MAVEN_HOME = tool name: 'Maven 3.9.3', type: 'maven'
        DOCKER_CREDENTIALS = 'docker-hub-creds' // Jenkins credential ID for Docker login
        DOCKER_IMAGE = 'your-dockerhub-username/ekart:latest'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Build & Test') {
            steps {
                withMaven(maven: 'Maven 3.9.3') {
                    sh 'mvn clean verify'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                // Make sure you have a Dependency-Check installation configured in Jenkins
                dependencyCheck odcInstallation: 'ODC', stopBuild: true, additionalArguments: '--format HTML'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS) {
                        def appImage = docker.build(DOCKER_IMAGE)
                        appImage.push()
                    }
                }
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
        always {
            // Clean workspace without wrapping in node
            cleanWs()
        }
    }
}
