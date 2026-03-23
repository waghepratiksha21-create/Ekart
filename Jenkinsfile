pipeline {
    agent any

    environment {
        // Update JDK to match your Jenkins Global Tool Configuration
        JAVA_HOME = tool name: 'JDK 21', type: 'jdk'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"
        MAVEN_HOME = tool name: 'Maven 3.9.3', type: 'maven'
        DOCKER_REGISTRY = 'your-docker-registry-url' // replace with your registry
        DOCKER_IMAGE = 'ekart-app'
        DOCKER_TAG = 'latest'
        NEXUS_REPO = 'docker-hosted' // replace with your Nexus repo
        DEPENDENCY_CHECK_FAIL_BUILD_ONCVSS = '7'
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Build & Test with Maven') {
            steps {
                withMaven(maven: 'Maven 3.9.3') {
                    sh 'mvn clean verify'
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--format HTML',
                                failBuildOnCVSS: env.DEPENDENCY_CHECK_FAIL_BUILD_ONCVSS
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    sh """
                        docker build -t ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG} .
                        echo $DOCKER_PASSWORD | docker login ${DOCKER_REGISTRY} --username $DOCKER_USERNAME --password-stdin
                        docker push ${DOCKER_REGISTRY}/${DOCKER_IMAGE}:${DOCKER_TAG}
                    """
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
            cleanWs()
        }
    }
}
