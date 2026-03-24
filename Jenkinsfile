pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar-token')
        DOCKERHUB_PWD = credentials('dockewrhub-pwd')
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/ygminds73/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('Unit Tests') {
            steps {
                catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                    sh 'mvn test'
                }
            }
        }

        stage('Build Package') {
            steps {
                sh 'mvn package -DskipTests=true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh 'mvn sonar:sonar -Dsonar.login=$SONAR_TOKEN'
                }
            }
        }

        stage('Dependency Check') {
            steps {
                sh '${tool "DC"}/bin/dependency-check.sh --project Ekart --scan .'
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh 'docker build -t youngminds73/ekart:latest -f docker/Dockerfile .'
                sh 'echo $DOCKERHUB_PWD | docker login -u youngminds73 --password-stdin'
                sh 'docker push youngminds73/ekart:latest'
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
