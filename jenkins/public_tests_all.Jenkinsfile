@Library('utils@dev') _

node {

    properties([
            copyArtifactPermission('${JOB_NAME},'+env.BRANCH_NAME),
    ]);

    stage('Checkout') {
        checkout scm;
    }

    def commitHash = sh(script: 'git rev-parse HEAD', returnStdout: true);
    commitHash = commitHash.replaceAll(/[^a-zA-Z0-9-]/, '');
    def DOCKER_IMAGE_TAG = "latest";
    stage('Build and Push Docker Image') {
        if (isDependencyInstallerChanged()) {
            DOCKER_IMAGE_TAG = pushCIImage(commitHash);
        }
    }
    def DOCKER_IMAGE = "openroad/flow-ubuntu22.04-dev:${DOCKER_IMAGE_TAG}";

    docker.image("${DOCKER_IMAGE}").inside('--user=root --privileged -v /var/run/docker.sock:/var/run/docker.sock') {
        stage('Build ORFS and Stash bins') {
            sh "git config --system --add safe.directory '*'";
            localBuild();
        }
    }

    stage('Run Tests') {
        Map tasks = [failFast: false];
        def test_slugs = getTestSlugs("dev");
        for (test in test_slugs) {
            def currentSlug = test; // copy needed to correctly pass args to runTests
            tasks["${test}"] = {
                node {
                    catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                        docker.image("${DOCKER_IMAGE}").inside('--user=root --privileged -v /var/run/docker.sock:/var/run/docker.sock') {
                            sh "git config --system --add safe.directory '*'";
                            checkout scm;
                            runTests(currentSlug);
                        }
                    }
                }
            }
        }
        parallel(tasks);
    }

    docker.image("${DOCKER_IMAGE}").inside('--user=root --privileged -v /var/run/docker.sock:/var/run/docker.sock') {
        sh "git config --system --add safe.directory '*'";
        stage('Upload Metadata') {
            uploadMetadata(commitHash);
        }
        stage('Generate Report Summary') {
            generateReport();
        }
        stage('Send Report') {
            sendEmail();
        }
    }

}
