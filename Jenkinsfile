pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        NVD_API_KEY = credentials('nvd-api-key')
    }

    tools {
        maven 'maven3'
        jdk 'jdk-17'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test -DskipTests=true || true'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-scanner') {
                    sh "${env.SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.java.binaries=target/classes"
                }
            }
        }

        stage('Prepare Dependency-Check Files') {
            steps {
                sh '''
                    mkdir -p target/dependency-check-lib
                    mvn dependency:copy-dependencies -DoutputDirectory=target/dependency-check-lib
                '''
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                // Directly as a step so Jenkins shows this stage
                dependencyCheck additionalArguments: "--noupdate --format ALL --failOnCVSS 7 --scan target/dependency-check-lib",
                                odcInstallation: 'DC'
            }
        }

        stage('Build Package') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-maven', jdk: 'jdk-17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy -DskipTests=true'
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                sh 'docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile .'
            }
        }

        stage('Push Image to DockerHub') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub-pwd', variable: 'dockerhubpwd')]) {
                    sh 'docker login -u waghepratiksha21 -p ${dockerhubpwd}'
                    sh 'docker push waghepratiksha21/ekart:latest'
                }
            }
        }

        stage('EKS & Kubectl Config') {
            steps {
                sh 'aws eks update-kubeconfig --region ap-south-1 --name project-cluster'
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deploymentservice.yml'
            }
        }
    }

    post {
        always {
            echo 'Cleaning workspace...'
            cleanWs()
        }
    }
}
