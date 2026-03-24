pipeline {
    agent any

    environment {
        SONAR_TOKEN = credentials('sonar-token')        // SonarQube token
        DOCKERHUB_PWD = credentials('dockewrhub-pwd')   // DockerHub PAT
        NVD_API_KEY = credentials('nvd-api-key')        // NVD API key (optional with --noupdate)
    }

    tools {
        maven 'maven3'
        jdk 'jdk17'
    }

    stages {

        stage('Checkout') {
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
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    sh 'mvn test || true'
                }
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
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
                catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                    script {
                        def dcPath = tool 'DC'
                        sh """
                            mkdir -p dependency-check-report
                            ${dcPath}/bin/dependency-check.sh \\
                                --project Ekart \\
                                --scan . \\
                                --noupdate \\
                                --format ALL \\
                                --failOnCVSS 7 \\
                                --out dependency-check-report
                        """
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/**', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    sh 'docker build -t waghepratiksha21/ekart:latest -f docker/Dockerfile .'
                    sh 'echo $DOCKERHUB_PWD | docker login -u waghepratiksha21 --password-stdin'
                    sh 'docker push waghepratiksha21/ekart:latest'
                }
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
