pipeline {
    agent any

    environment {
        MAVEN_HOME = tool 'maven3'
        JDK_HOME = tool 'jdk17'
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_IMAGE = "waghepratiksha21/ekart"
        DOCKER_TAG = "latest"
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn clean compile"
            }
        }

        stage('Unit Tests') {
            steps {
                // Run tests but don't fail the pipeline if tests fail
                sh "${MAVEN_HOME}/bin/mvn test jacoco:report || true"
            }
        }

        stage('SonarQube Analysis') {
            steps {
                // Use only the Jenkins SonarQube configuration and token
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.tests=src/test/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                        -Dsonar.scm.provider=git
                    """
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: "--nvdApiKey=${NVD_API_KEY}",
                                odcInstallation: 'DC'
            }
        }

        stage('Build Package') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn package -DskipTests=true"
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', maven: 'maven3', jdk: 'jdk17') {
                    sh "${MAVEN_HOME}/bin/mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} -f docker/Dockerfile ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([string(credentialsId: 'dockewrhub-pwd', variable: 'DOCKER_HUB_PWD')]) {
                    sh "docker login -u waghepratiksha21 -p ${DOCKER_HUB_PWD}"
                    sh "docker push ${DOCKER_IMAGE}:${DOCKER_TAG}"
                }
            }
        }

        stage('Configure EKS') {
            steps {
                sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }

        stage('Create LoadBalancer for Deployment') {
            steps {
                script {
                    sh '''
                        kubectl expose deployment ekart-deployment \
                            --type=LoadBalancer \
                            --name=ekart-service \
                            --port=80 \
                            --target-port=8080 || echo "Service already exists"

                        echo "Waiting for LoadBalancer IP..."
                        for i in {1..30}; do
                            LB_IP=$(kubectl get svc ekart-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
                            if [ ! -z "$LB_IP" ]; then
                                echo "LoadBalancer available at: $LB_IP"
                                break
                            fi
                            sleep 10
                        done
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace...'
            cleanWs()
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the logs above for errors.'
        }
    }
}
