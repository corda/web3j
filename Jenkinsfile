#!groovy
pipeline {
    agent {
        docker {
            // Our custom docker image
            image "build-zulu-openjdk:11"
            label "standard"
            registryUrl 'https://engineering-docker.software.r3.com/'
            registryCredentialsId 'artifactory-credentials'
            // Used to mount storage from the host as a volume to persist the cache between builds
            args '-v /tmp:/host_tmp'
            alwaysPull true
        }
    }
    environment {
        ARTIFACTORY_CREDENTIALS = credentials('artifactory-credentials')
        ARTIFACTORY_BUILD_NAME = "Web3j artifact"
        ARTIFACTORY_SERVER = "software.r3.com"
        ARTIFACTORY_SERVER_ID = 'R3-Artifactory'
        GRADLE_USER_HOME = "/host_tmp/gradle"
        CORDA_ARTIFACTORY_PASSWORD = "${env.ARTIFACTORY_CREDENTIALS_PSW}"
        CORDA_ARTIFACTORY_USERNAME = "${env.ARTIFACTORY_CREDENTIALS_USR}"
        CORDA_ARTIFACTORY_REPOKEY =  "${isReleaseTag() ? 'corda-dependencies' : 'corda-dependencies-dev'}"
    }
    options {
        ansiColor('xterm')
        buildDiscarder logRotator(daysToKeepStr: '14')
        disableConcurrentBuilds()
        timestamps()
        timeout(activity: true, time: 10)
    }
    stages {
        stage('Build') {
            steps {
                sh './gradlew assemble'
            }
        }
        stage('Publish') {
            steps {
                echo "Creating Artifactory server ${env.ARTIFACTORY_SERVER_ID}"
                rtServer(
                        id: "${env.ARTIFACTORY_SERVER_ID}",
                        url: "https://${env.ARTIFACTORY_SERVER}/artifactory",
                        credentialsId: 'artifactory-credentials'
                )
                rtBuildInfo(
                        captureEnv: true,
                        buildName: env.ARTIFACTORY_BUILD_NAME
                )
               rtGradleDeployer(
                        id: 'deployer',
                        serverId: 'R3-Artifactory',
                        repo: '${CORDA_ARTIFACTORY_REPOKEY}'
               )
               rtGradleRun(
                        usesPlugin: true,
                        useWrapper: true,
                        switches: '-s --info -x signMavenPublication',
                        tasks: 'artifactoryPublish',
                        deployerId: 'deployer',
                        buildName: env.ARTIFACTORY_BUILD_NAME
               )
                echo 'Publishing build info to Artifactory server'
                rtPublishBuildInfo(
                        serverId: env.ARTIFACTORY_SERVER_ID,
                        buildName: env.ARTIFACTORY_BUILD_NAME
                )
            }
        }
    }
}


def isReleaseTag() {
    return (env.TAG_NAME =~ /^release-.*$/)
}
