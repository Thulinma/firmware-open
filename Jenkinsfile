#!/usr/bin/env groovy

// Required plugins:
// - Jenkins Core
// - Pipeline (https://plugins.jenkins.io/workflow-aggregator/)
// - GitHub (https://plugins.jenkins.io/github/)
// - AnsiColor (https://plugins.jenkins.io/ansicolor/)

def all_models = 'addw2 bonw14 darp5 darp6 darp7 galp3-c galp4 galp5 gaze15 gaze16-3050 gaze16-3060 gaze16-3060-b lemp9 lemp10 oryp6 oryp7 oryp8'

def getCommitSha() {
  sh "git rev-parse HEAD > .git/current-commit"
  return readFile(".git/current-commit").trim()
}

void setBuildStatus(String message, String state) {
    commitSha = getCommitSha()

    step([
        $class:  'GitHubCommitStatusSetter',
        reposSource: [$class: "ManuallyEnteredRepositorySource", url: "https://github.com/system76/firmware-open"],
        contextSource: [$class: "ManuallyEnteredCommitContextSource", context: "ci/jenkins/build-status"],
        commitShaSource: [$class: "ManuallyEnteredShaSource", sha: commitSha],
        errorHandlers: [[$class: "ChangingBuildStatusErrorHandler", result: "UNSTABLE"]],
        statusResultSource: [ $class: "ConditionalStatusResultSource", results: [[$class: "AnyBuildResult", message: message, state: state]] ]
    ]);
}

pipeline {
    agent {
        //label 'warp.pop-os.org'
        label 'firmware'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '16', artifactNumToKeepStr: '1'))
        disableConcurrentBuilds()
        timeout(time: 1, unit: 'HOURS')
        timestamps()
        ansiColor('xterm')
    }

    parameters {
        string(name: 'MODELS', defaultValue: "$all_models", description: 'Space separated list of models to build', trim: true)
        string(name: 'GIT_BRANCH', defaultValue: '', description: 'Git branch or revision to build (master, for example)', trim: true)
    }

    // TODO: Set up webhook so trigger isn't needed.
    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Prepare') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${GIT_BRANCH}"]],
                    extensions: [[
                        $class: 'SubmoduleOption',
                        disableSubmodules: false,
                        parentCredentials: true,
                        recursiveSubmodules: true,
                        reference: '',
                        trackingSubmodules: false
                    ]],
                    userRemoteConfigs: [[url: 'https://github.com/system76/firmware-open']]
                ])

                setBuildStatus("Pending", "PENDING")

                sh """#!/bin/bash
                    # Install dependencies
                    #./scripts/deps.sh
                    . "${HOME}/.cargo/env"

                    # Reset
                    git submodule update --init --recursive --checkout
                    git reset --hard
                    git submodule foreach --recursive git reset --hard

                    # Clean
                    git clean -dffx
                    git submodule foreach --recursive git clean -dff

                    # EDK2 builds fail if file paths in INFs change from what's in the build cache
                    pushd edk2; git clean -dffx; popd
                """
            }
        }
        stage('Build') {
            steps {
                // The workspace is reused, so must build models sequentially.
                script {
                    def list = params.MODELS.tokenize()
                    list.each { model ->
                        stage(model) {
                            sh """#!/bin/bash
                                . "${HOME}/.cargo/env"
                                # WORSKSPACE is set by Jenkins, but EDK2 uses it
                                env --unset=WORKSPACE \
                                ./scripts/build.sh "${model}"
                            """
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'build/*/*', allowEmptyArchive: true
        }
        success {
            setBuildStatus("Successful", "SUCCESS")
        }
        failure {
            setBuildStatus("Failed", "FAILURE")
        }
    }
}
