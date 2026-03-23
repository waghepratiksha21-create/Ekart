pipeline {
    agent any   // Use any available agent

    // Tool configurations
    environment {
        JAVA_HOME = tool name: 'JDK 21', type: 'jdk'  // Make sure JDK 21 exists in Jenkins
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        MAVEN_HOME = tool name: 'Maven 3.9.3', type: 'maven'
        DOCKER_CREDENTIALS = 'docker-hub-creds'      // Docker credentials ID in Jenkins
        DOCKER_IMAGE = 'your-dockerhub-username/ekart:latest'
    }

    options {
        skipDefaultCheckout(true)  // We’ll do checkout explicitly
        timeout(time: 60, unit: 'MINUTES')
    }

    stages {

        stage('Checkout SCM') {
            steps {
                // Checkout source code
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
                // Dependency-Check plugin must be installed, ODC installation configured in Jenkins
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

    } // end stages

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()  // Safe in Declarative Pipeline post block
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
