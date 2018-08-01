#!/usr/bin/env groovy

node {

    APP_TYPE = "java-proj-docker"

    // Clean workspace
    stage('Clean workspace') {
        cleanWs()
    }

    // Check out the repository
    stage('Checkout') {
        checkout scm
        // Get commit ID
        sh "git rev-parse HEAD | tail -c 8 > var_short_commit-id"
        // Stash vars
        stash includes: "var_*", name: "var-stash"
        // Abort build if this is commit message has been done by gitflow
        result = sh (script: "git log -1 --pretty=%B | grep -i 'updating develop poms \\|updating poms for '", returnStatus: true)
        if (result == 0) {
            currentBuild.result = 'ABORTED'
            error('Aborting build because it is a gitflow commit')
        }
    }

    // Get branch and commit id
    unstash "var-stash"
    def short_commit_id = readFile('var_short_commit-id').trim()

    println "Git branch = ${BRANCH_NAME}, Git commit id: ${short_commit_id}"

    // If we're master, verify develop branch and then do a Gitflow release followed by Sonar report
    if ("${BRANCH_NAME}" == "master") {
        // Release to Artifactory
        stage('Verify and Gitflow Release to Artifactory') {
                withMaven(
                        maven: 'maven',
                        jdk: 'java'
                ) {
                    // Run the verify on develop
                    sh "git checkout develop"

                    sh "mvn clean verify -Pintegration-test"

                    // Go back to master
                    sh "git checkout ${BRANCH_NAME}"

                    sh "mvn jgitflow:release-start " +
                            "-DallowUntracked=true " +
                            "-DallowSnapshots " +
                            "-Dusername=${USERNAME} " +
                            "-Dpassword=${PASSWORD}"

                    sh "mvn jgitflow:release-finish " +
                            "-Psonar,integration-test " +
                            "-DallowUntracked=true " +
                            "-Dmaven.javadoc.skip=true " +
                            "-DallowSnapshots " +
                            "-Dsonar.analysis.mode=publish " +
                            "-Dusername=${USERNAME} " +
                            "-Dpassword=${PASSWORD}"
                }
        }
    }

    // If we're develop, verify only
    if ("${BRANCH_NAME}" == "develop") {
        stage('Clean Verify') {
            withMaven(
                    maven: 'maven',
                    jdk: 'java'
            ) {
                sh "mvn clean verify -Pintegration-test"
            }
        }
    }

    // Anything else, verify and Sonar check commenting PR if it exists
    if ("${BRANCH_NAME}" != "master" && "${BRANCH_NAME}" != "develop") {
        stage('Clean verify') {
            withMaven(
                    maven: 'maven',
                    jdk: 'java'
            ) {
                sh "mvn clean verify"
            }
        }
    }

    if ("${BRANCH_NAME}" == "master" || "${BRANCH_NAME}" == "develop") {
        // Build Docker image, push image to ECR
        stage('Create and push Docker container') {
            withMaven(
                    maven: 'maven',
                    jdk: 'java'
            ) {
                sh "mvn docker:build -DpushImage -Dgit.commit.id.abbrev=${short_commit_id}"
            }
        }
    }
}

