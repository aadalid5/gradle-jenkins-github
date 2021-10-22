pipeline {
    agent any

    tools {
        jdk 'openjdk-11'
    }

    environment {
        commitID = getCommitID()
        version = getVersion()
        artifact = 'hbr-ms-user'
        BU = 'hbrg'
        env = getEnvironment()
    }

    stages {
        stage('Build') {
            steps {
                sh "echo 'Build...' "
                sh "whoami"
                sh "pwd"
                sh "cat /etc/*release"
                sh "./gradlew clean"
                sshagent(credentials: ['e45c738f-4214-4296-96a0-e53307dab985']) {
                    sh "git config --list "
                }
            }
        }

        stage('Deploy QA') {
//            when {
//                expression {
//                    return isReleaseBuild()
//                }
//            }

            steps {
                echo "Deploying QA-${artifact}-${version}-${commitID}-${env.BUILD_NUMBER}"

                sshagent(credentials: ['e45c738f-4214-4296-96a0-e53307dab985']) {
                    sh "git reset --hard origin/master"

                    // update the versions
                    sh "./gradlew release -Prelease.useAutomaticVersion=true -Prelease.releaseVersion=${version} -Prelease.newVersion=0.0.4-SNAPSHOT"

                    // push the tag
                    echo "Tagging Release ${version}"
                    sh "git push --tags"

                    // update the master branch
                    sh "git push origin HEAD:master"
                }
            }
        }

    }
}

def isPrAgainstMaster() {
    return env.CHANGE_TARGET == "master" && env.CHANGE_ID
}

def isReleaseBuild() {
    return env.BRANCH_NAME == 'master'
}

def buildFailed() {
    return currentBuild.result == 'UNSTABLE' || currentBuild.result == 'FAILURE'
}

// parse the git info for the commit ID
def getCommitID() {
    sh 'git rev-parse HEAD > .git/commitID'
    def commitID = readFile('.git/commitID').trim()
    sh 'rm .git/commitID'
    commitID = commitID[0..6]
    return commitID
}

def notifySlack(String buildStatus = 'STARTING') {
    buildStatus = buildStatus ?: 'SUCCESS'

    def color
    groovy.lang.GString msg = "${env.JOB_NAME} - #${env.BUILD_NUMBER} ${version} ${commitID} ${buildStatus} "
    def seconds = currentBuild.duration / 1000

    if (buildStatus == 'STARTING' || buildStatus == 'NOT_BUILT') {
        color = '#D4DADF'
    } else if (buildStatus == 'SUCCESS') {
        color = '#BDFFC3'
    } else if (buildStatus == 'UNSTABLE') {
        color = '#FFFE89'
    } else {
        color = '#FF9FA1'
    }

    if (buildStatus != 'STARTING') {
        msg += " after ${seconds} seconds (<${env.BUILD_URL}|Open>)\n" +
                "(<${env.BUILD_URL}/Build_Reports/|Build Reports>)"
    }

    slackSend(color: color, message: msg)
}

def buildDocker(String BU, String appName, String tag, String springProfile) {
    sh "sudo docker build -t ${BU}/${appName}:${tag} --build-arg VERSION=${version} --build-arg SPRING_PROFILE=${springProfile} ."
}

def tagPushDocker(String BU, String appName, String tag) {
    sh "aws ecr get-login-password --region us-east-1 \
  | sudo docker login --username AWS --password-stdin 964020329682.dkr.ecr.us-east-1.amazonaws.com/${BU}/${appName}"
    sh "sudo docker tag ${BU}/${appName}:${tag} 964020329682.dkr.ecr.us-east-1.amazonaws.com/${BU}/${appName}:${tag}"
    sh "sudo docker push 964020329682.dkr.ecr.us-east-1.amazonaws.com/${BU}/${appName}:${tag}"
    sh "sudo docker tag ${BU}/${appName}:${tag} 964020329682.dkr.ecr.us-east-1.amazonaws.com/${BU}/${appName}:latest"
    sh "sudo docker push 964020329682.dkr.ecr.us-east-1.amazonaws.com/${BU}/${appName}:latest"
}

def checkSkip() {
    // check to see if we have a Jenkins initiated build and skip
    env.SKIP = "false"
    result = sh(script: "git log -1 | grep '.*\\[jenkins-ci\\].*'", returnStatus: true)
    if (result == 0) {
        echo "'[jenkins-ci]' found in git commit message. Aborting."
        env.SKIP = "true"
        currentBuild.result = 'SUCCESS'
    }
}

def shouldSkip() {
    return (env.SKIP == "true")
}

def getEnvironment() {
    if (isReleaseBuild()) {
        return "prod"
    } else if (isPrAgainstMaster()) {
        return "QA"
    } else {
        return "UNK"
    }
}

def getVersion() {
    version = sh script: "./gradlew -q getVersion", returnStdout: true
    if (isReleaseBuild()) {
        version = version.replace("-SNAPSHOT", "")
    }
    return version
}