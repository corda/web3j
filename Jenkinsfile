#!groovy

import groovy.transform.Field

@Field
Map defaults = [
    /*
     * the name of a build in Artifactory
     */
    artifactoryBuildName: 'Web3j artifact',
    /*
     * the address of Artifactory server to publish build info to
     */
    artifactoryServer: 'software.r3.com',
    /*
     * the name of Artifactory Docker repository to push build image to
     * when building from a release tag
     */
    aritfactoryReleaseRepo: null,
]

def call(Map config) {
    if (config == null) {
        config = defaults
    } else {
        config = defaults + config
    }

    assert config.artifactoryReleaseRepo :'The artifactoryReleaseRepo parameter cannot be empty'

pipeline {
        agent {
            kubernetes {
                cloud 'kubernetes'
                inheritFrom 'default'
                yaml '''\
                    apiVersion: v1
                    kind: Pod
                    spec:
                        containers:
                          - name: docker
                            image: docker:latest
                            command:
                                - sleep
                            args:
                                - infinity
                            env:
                            - name: DOCKER_HOST
                              value: tcp://localhost:2375
                          - name: dind
                            image: docker:18.05-dind
                            securityContext:
                              privileged: true
                            volumeMounts:
                              - name: dind-storage
                                mountPath: /var/lib/docker
                        volumes:
                          - name: dind-storage
                            emptyDir: {}
                '''.stripIndent()
                defaultContainer 'docker'
            }
        }
        environment {
            ARTIFACTORY_BUILD_NAME = "${config.artifactoryBuildName}"
            ARTIFACTORY_SERVER = "${config.artifactoryServer}"
            ARTIFACTORY_REPO_NAME = "${artifactoryRepoName}"
            ARTIFACTORY_SERVER_ID = 'R3-Artifactory'
            DOCKER_TAG = "${getDockerTag()}"
            FULL_IMAGE_NAME = "${artifactoryRepoName}.${config.artifactoryServer}/${config.imageName}:${env.DOCKER_TAG}"
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
                    echo "Pushing '${FULL_IMAGE_NAME}' image to Artifactory Docker registry (${env.ARTIFACTORY_REPO_NAME})"
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
                    echo 'Publishing build info to Artifactory server'
                    rtPublishBuildInfo(
                            serverId: env.ARTIFACTORY_SERVER_ID,
                            buildName: env.ARTIFACTORY_BUILD_NAME
                    )
                }
            }
        }
    }
}