pipeline {
    agent any
    tools { maven 'Maven3' }
    triggers {
        githubPush()
        pollSCM('H/2 * * * *')
    }

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
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                      scp -o StrictHostKeyChecking=no target/myapp.jar ubuntu@13.127.142.181:/home/ubuntu/myapp.jar
                      ssh -o StrictHostKeyChecking=no ubuntu@13.127.142.181 "java -jar /home/ubuntu/myapp.jar"
                    '''
                }
            }
        }
        
    post {
        success { echo 'Build, tests and deployment succeeded' }
        failure { echo 'Pipeline failed: deployment skipped' }
    }
}
