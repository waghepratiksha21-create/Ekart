pipeline {
    agent any

    environment {
        MAVEN_HOME = tool 'maven3'
        JDK_HOME = tool 'jdk8' // Changed to existing JDK
        DOCKER_IMAGE = "ekart-app"
        DOCKER_TAG = "latest"
        SONAR_HOST = "http://13.233.125.170:9000"
        SONAR_TOKEN = credentials('sonarqube-token')
    }

    tools {
        maven 'maven3'
        jdk 'jdk8' // Use installed JDK
    }

    options {
        skipDefaultCheckout(true)
        timestamps()
        // ansiColor('xterm')  <-- remove or install plugin
    }

    stages {
        stage('Checkout') { steps { git branch: 'master', url: 'https://github.com/waghepratiksha21-create/Ekart.git' } }
        stage('Compile') { steps { sh "${MAVEN_HOME}/bin/mvn clean compile" } }
        stage('Unit Tests') { steps { sh "${MAVEN_HOME}/bin/mvn test jacoco:report" } }
        stage('SonarQube Analysis') { 
            steps {
                sh """
                    docker run --rm -v "\$PWD":/usr/src \
                        -e SONAR_TOKEN=${SONAR_TOKEN} \
                        sonarsource/sonar-scanner-cli:latest \
                        sonar-scanner \
                        -Dsonar.projectKey=EKART \
                        -Dsonar.projectName=EKART \
                        -Dsonar.sources=/usr/src/src/main/java \
                        -Dsonar.tests=/usr/src/src/test/java \
                        -Dsonar.java.binaries=/usr/src/target/classes \
                        -Dsonar.coverage.jacoco.xmlReportPaths=/usr/src/target/site/jacoco/jacoco.xml \
                        -Dsonar.host.url=${SONAR_HOST} \
                        -Dsonar.token=${SONAR_TOKEN} \
                        -Dsonar.scm.provider=git
                """
            }
        }
    }
}
