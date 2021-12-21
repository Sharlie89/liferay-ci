#!/usr/bin/env groovy
pipeline {
    agent any
    // global env variables
    environment {
        EMAIL_RECIPIENTS = 'carlos.abarba@babel.es'
        PATH = "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/bin"
    }
    stages {
        stage ('Checkout'){
          steps{
            checkout scm
          }
        }
        stage ('Install modules'){
            steps{
                dir('hola-mundo'){
                    sh '''
                        npm install
                    '''
                   /* npm install --verbose -d */
                }
            }
        }
        stage ('Angular build'){
            steps{
                dir('hola-mundo'){
                    sh '''
                       ng build
                    '''
                }
            }
        }
        stage ('LifeRay build'){
            steps{
                dir('hola-mundo'){
                    sh '''
                       npm run build:liferay
                    '''
                }
            }
        }
        stage ('Publish build info') {
            steps {
                rtPublishBuildInfo (
                    serverId: "artifactory"
                )
            }
        }
        stage ('Publish build artifact') {
            steps {
                sh ''' ls -l ${WORKSPACE}/hola-mundo '''
                rtUpload(
                    serverId: "artifactory",
                    spec: """{
                            "files": [
                                    {
                                        "pattern": "${WORKSPACE}/hola-mundo/*.gz",
                                        "target": "libs-snapshot-local"
                                    }
                                ]
                            }"""
                )
            }
        }
/*        stage('Upload to Artifactory') {
           steps {
              script {
                 def server = Artifactory.server 'artifactory'
                 def uploadSpec = """{
                    "files": [{
                       "pattern": "test/",
                       "target": "test/"
                    }]
                 }"""
                 server.upload(uploadSpec)
               }
            }
        } */
}
    post {
        // Always runs. And it runs before any of the other post conditions.
        always {
            // Let's wipe out the workspace before we finish!
            deleteDir()
        }
/*        success {
            sendEmail("Successful");
        }
        unstable {
            sendEmail("Unstable");
        }
        failure {
            sendEmail("Failed");
        }*/
    }

// The options directive is for configuration that applies to the whole job.
    options {
        // For example, we'd like to make sure we only keep 10 builds at a time, so
        // we don't fill up our storage!
        buildDiscarder(logRotator(numToKeepStr: '5'))

        // And we'd really like to be sure that this build doesn't hang forever, so
        // let's time it out after an hour.
        timeout(time: 25, unit: 'MINUTES')
    }

}
def developmentArtifactVersion = ''
def releasedVersion = ''
// get change log to be send over the mail
@NonCPS
def getChangeString() {
    MAX_MSG_LEN = 100
    def changeString = ""

    echo "Gathering SCM changes"
    def changeLogSets = currentBuild.changeSets
    for (int i = 0; i < changeLogSets.size(); i++) {
        def entries = changeLogSets[i].items
        for (int j = 0; j < entries.length; j++) {
            def entry = entries[j]
            truncated_msg = entry.msg.take(MAX_MSG_LEN)
            changeString += " - ${truncated_msg} [${entry.author}]\n"
        }
    }

    if (!changeString) {
        changeString = " - No new changes"
    }
    return changeString
}

def sendEmail(status) {
    mail(
            to: "$EMAIL_RECIPIENTS",
            subject: "Build $BUILD_NUMBER - " + status + " (${currentBuild.fullDisplayName})",
            body: "Changes:\n " + getChangeString() + "\n\n Check console output at: $BUILD_URL/console" + "\n")
}

def getDevVersion() {
    def gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    def versionNumber;
    if (gitCommit == null) {
        versionNumber = env.BUILD_NUMBER;
    } else {
        versionNumber = gitCommit.take(8);
    }
    print 'build  versions...'
    print versionNumber
    return versionNumber
}

def getReleaseVersion() {
    def pom = readMavenPom file: 'pom.xml'
    def gitCommit = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
    def versionNumber;
    if (gitCommit == null) {
        versionNumber = env.BUILD_NUMBER;
    } else {
        versionNumber = gitCommit.take(8);
    }
    return pom.version.replace("-SNAPSHOT", ".${versionNumber}")
}