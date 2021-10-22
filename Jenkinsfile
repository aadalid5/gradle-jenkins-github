pipeline {
    agent any
    environment {
        CI = 'true'
    }
    stages {
        stage('Build') {
            steps {
                sh "echo 'hello pipeline' "
                sh "whoami"
                sh "pwd"
                sh "cat /etc/*release"
            }
        }
    }
}
