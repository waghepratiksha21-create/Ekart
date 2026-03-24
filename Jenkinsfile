pipeline {
    agent any

    // ------------------ Environment ------------------
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        MAVEN_HOME   = tool 'maven3'
        JAVA_HOME    = tool 'jdk17'
        NVD_API_KEY  = credentials('nvd-api-key')          // Dependency-Check secret
        SONAR_TOKEN  = credentials('sonar-token')         // SonarQube token
        DOCKERHUB_PWD = credentials('dockewrhub-pwd')     // DockerHub token
        DOCKERHUB_USER = 'your-dockerhub-username'        // Replace with your DockerHub username
        IMAGE_NAME    = "shopping-cart"
        IMAGE_TAG     = "${env.BUILD_NUMBER}"
        KUBE_CONFIG   = credentials('eks-kubeconfig')     // Optional: if using kubeconfig
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
        ansiColor('xterm')
    }

    stages {

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git'
            }
        }

        stage('Compile & Package') {
            steps {
                withEnv(["JAVA_HOME=${env.JAVA_HOME}", "PATH+JAVA=${env.JAVA_HOME}/bin"]) {
                    sh "${MAVEN_HOME}/bin/mvn clean package -DskipTests=true"
                }
            }
        }

        stage('Unit Tests') {
            steps {
                withEnv(["JAVA_HOME=${env.JAVA_HOME}", "PATH+JAVA=${env.JAVA_HOME}/bin"]) {
                    sh "${MAVEN_HOME}/bin/mvn test"
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withEnv(["JAVA_HOME=${env.JAVA_HOME}", "PATH+JAVA=${env.JAVA_HOME}/bin"]) {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                        sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=shopping-cart \
                        -Dsonar.sources=src/main/java \
                        -Dsonar.java.binaries=target/classes \
                        -Dsonar.host.url=http://13.233.125.170:9000 \
                        -Dsonar.login=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withEnv(["JAVA_HOME=${env.JAVA_HOME}", "PATH+JAVA=${env.JAVA_HOME}/bin"]) {
                    sh """
                    dependency-check.sh --project shopping-cart \
                    --scan ./ \
                    --format HTML \
                    --out dependency-check-report \
                    --enableExperimental \
                    --nvdApiKey ${NVD_API_KEY}
                    """
                }
            }
        }

        stage('Deploy to Nexus') {
            steps {
                withEnv(["JAVA_HOME=${env.JAVA_HOME}", "PATH+JAVA=${env.JAVA_HOME}/bin"]) {
                    sh "${MAVEN_HOME}/bin/mvn deploy -DskipTests=true"
                }
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    sh "docker build -t ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh "echo ${DOCKERHUB_PWD} | docker login -u ${DOCKERHUB_USER} --password-stdin"
                    sh "docker push ${DOCKERHUB_USER}/${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }

        stage('Configure EKS & Deploy to Kubernetes') {
            steps {
                script {
                    // Optional: if you have kubeconfig as credential
                    sh "export KUBECONFIG=${KUBE_CONFIG}"
                    sh "kubectl apply -f k8s/deployment.yaml"
                    sh "kubectl apply -f k8s/service.yaml"
                }
            }
        }

        stage('Post Actions') {
            steps {
                echo "Pipeline completed successfully!"
            }
        }
    }

    post {
        success {
            echo "Build, Scan, Deploy pipeline SUCCESS!"
        }
        failure {
            echo "Pipeline FAILED. Check logs!"
        }
        always {
            cleanWs()
        }
    }
}
