pipeline {
    agent any
    tools { maven 'Maven3' }
    triggers { githubPush() }

    stages {
        stage('Checkout') { steps { checkout scm } }

        stage('Build') { steps { sh 'mvn clean compile' } }

        stage('Test') {
            steps { sh 'mvn test' }
            post { always { junit 'target/surefire-reports/*.xml' } }
        }

        stage('Package') {
            steps { sh 'mvn package -DskipTests' }
            post { success { archiveArtifacts 'target/myapp.jar' } }
        }

        stage('Deploy to Staging') {
            steps {
                sh '''
                  mkdir -p /var/jenkins_home/staging
                  cp target/myapp.jar /var/jenkins_home/staging/myapp.jar
                  java -jar /var/jenkins_home/staging/myapp.jar
                '''
            }
        }
    }

    post {
        success { echo 'Build, tests and deployment succeeded' }
        failure { echo 'Pipeline failed: deployment skipped' }
    }
}
