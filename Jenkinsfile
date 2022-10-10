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
        ARTIFACTORY_BUILD_NAME = "Web3j artifact"
        ARTIFACTORY_SERVER = "software.r3.com"
        ARTIFACTORY_REPO_NAME = "corda-dependency"
        ARTIFACTORY_SERVER_ID = 'R3-Artifactory'
        FULL_IMAGE_NAME = "${env.ARTIFACTORY_REPO_NAME}.${env.ARTIFACTORY_SERVER}/web3j-fork:${env.GIT_COMMIT.substring(0, 7)}"
    }
    options {
        ansiColor('xterm')
        buildDiscarder logRotator(daysToKeepStr: '14')
        disableConcurrentBuilds()
        timestamps()
        timeout(activity: true, time: 10)
    }
    stages {
        stage('Prepare') {
            steps {
                sh 'apk add --update openjdk8-jre'
            }
        }
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
                echo "Pushing '${env.FULL_IMAGE_NAME}' image to Artifactory Docker registry (${env.ARTIFACTORY_REPO_NAME})"
               rtGradleDeployer(
                        id: 'deployer',
                        serverId: 'R3-Artifactory',
                        repo: 'corda-dependency'
               )
               rtGradleRun(
                        usesPlugin: true,
                        useWrapper: true,
                        switches: '-s --info',
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