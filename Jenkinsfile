pipeline {
    agent any

    environment {
        JAVA_HOME = tool name: 'JDK 21', type: 'jdk'  // Must match Jenkins Global Tool name
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        MAVEN_HOME = tool name: 'Maven 3.9.3', type: 'maven'
        DOCKER_CREDENTIALS = 'docker-hub-creds'      // Set in Jenkins credentials
        DOCKER_IMAGE = 'your-dockerhub-username/ekart:latest'
    }

    options {
        timeout(time: 60, unit: 'MINUTES')
        skipDefaultCheckout(true)
    }

    stages {

        stage('Checkout SCM') {
            steps {
                node {  // Ensures workspace context
                    checkout([$class: 'GitSCM',
                        branches: [[name: '*/master']],
                        userRemoteConfigs: [[url: 'https://github.com/waghepratiksha21-create/Ekart.git']]
                    ])
                }
            }
        }

        stage('Build & Test') {
            steps {
                node {
                    withMaven(maven: 'Maven 3.9.3') {
                        sh 'mvn clean verify'
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                node {
                    dependencyCheck odcInstallation: 'ODC', stopBuild: true, additionalArguments: '--format HTML'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                node {
                    script {
                        docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS) {
                            def appImage = docker.build(DOCKER_IMAGE)
                            appImage.push()
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace...'
            script {
                node {  // Node required for cleanWs
                    cleanWs()
                }
            }
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
