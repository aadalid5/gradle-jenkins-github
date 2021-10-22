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

                sshagent(credentials: ['e45c738f-4214-4296-96a0-e53307dab985']) {
                    sh "git config --list "
                }
            }
        }
    }
}
