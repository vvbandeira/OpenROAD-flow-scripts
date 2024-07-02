@Library('utils@dev') _

node {

    properties([copyArtifactPermission('${JOB_NAME},'+env.BRANCH_NAME)]);

    def commitHash;
    stage('Checkout') {
        checkout scm;
        commitHash = sh(script: 'git rev-parse HEAD', returnStdout: true);
        commitHash = commitHash.replaceAll(/[^a-zA-Z0-9-]/, '');
    }

    def DOCKER_TAG = "latest";

    stage('Build and Push Docker Image') {
        if (env.BRANCH_NAME == 'master') {
            DOCKER_TAG = sh(script: './etc/DockerTag.sh -master', returnStdout: true);
        } else {
            DOCKER_TAG = sh(script: './etc/DockerTag.sh -dev', returnStdout: true);
        }
        if (!dockerImageExists(DOCKER_TAG)) {
            dockerPush('dev', DOCKER_TAG);
        }
        if (env.BRANCH_NAME == 'master') {
            dockerPush('master', DOCKER_TAG);
        }
    }
    def DOCKER_IMAGE = "openroad/flow-ubuntu22.04-dev:${DOCKER_TAG}";

    stage('Build ORFS and Stash bins') {
        localBuild(DOCKER_IMAGE);
    }

    stage('Run Tests') {
        runTests('dev');
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
