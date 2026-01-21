@Library(value = 'shared-pipeline-libs@1.4.23', changelog = false)
import groovy.transform.Field

/*
 * Constants
 */

@Field
private static final String PROJECT_NAME = "ort"
@Field
private static final String PROJECT_PREFIX = 'cybsec'
@Field
private static final String NEXUS_CREDENTIAL_ID = 'dockerDeployment'
@Field
private String SANITIZED_BRANCH_NAME;
@Field
private boolean IS_PIPELINE_STARTED_BY_USER;
@Field
private String branchToScan = ""
IS_PIPELINE_STARTED_BY_USER = semvoxAdapter.isPipelineStartedByUser()
// nexus has some character restrictions on repo names etc.
SANITIZED_BRANCH_NAME = semvoxAdapter.getSanitizedBranchName();

node(label: 'linux') {
    properties([
            buildDiscarder(logRotator(artifactDaysToKeepStr: '', artifactNumToKeepStr: '', daysToKeepStr: '30', numToKeepStr: '10')),
            parameters([
                    string(name: 'TAG_OR_BRANCH', defaultValue: "test", description: """Branch or tag to checkout for the ort repository\n
                                    Will be used with the build command as:\n
                                    --build-arg ORT_VERSION=params.TAG_OR_BRANCH""")
            ]),
            disableConcurrentBuilds()
    ])
    
    if(params.TAG_OR_BRANCH.equals("")) {
        semvoxAdapter.error("please provide a tag for build and deploy image")
    }
    if(!IS_PIPELINE_STARTED_BY_USER) {
        semvoxAdapter.info("Please trigger the build manually to build and deploy the image")
        return
    }

    stage("Cleanup Workspace") {
        cleanWs()
    }

    try {
        currentBuild.result = "SUCCESS"
        // branchTagToScan=params.TAG_OR_BRANCH
        stage("Checkout SCM") {
            checkout scm
            git(url: "ssh://git@bitbucket.intern.semvox.de/cysec/ort.git", branch: env.BRANCH_NAME, credentialsId: scm.getUserRemoteConfigs()[0].getCredentialsId())
        }

        // stage("GetORTSources") {
            // sh "git checkout $branchTagToScan"
            // semvoxAdapter.info("checkout the tag to to be build: ${branchTagToScan}")
        // }

        stage("Build image") {
            def dockerImage

            env.DOCKER_BUILDKIT = 1
            def tag = params.TAG_OR_BRANCH
            dockerImage = docker.build("${PROJECT_PREFIX}/${PROJECT_NAME}:${tag}", "--build-arg ORT_VERSION=${params.TAG_OR_BRANCH} --build-arg USERNAME=jenkins --build-arg USER_ID=1001 .")

            semvoxAdapter.info("Push image")
            docker.withRegistry('https://repo.semvox.ai:5000', NEXUS_CREDENTIAL_ID) {
                dockerImage.push()
            }
        }
    }
    catch (e) {
        currentBuild.result = "FAILURE"
        throw (e)
    }
    finally {
        stage("E-Mail notification") {
            emailext(
                    mimeType: 'text/html',
                    from: 'noreply@semvox.de',
                    recipientProviders: [requestor()],
                    replyTo: 'noreply@semvox.de',
                    subject: "${currentBuild.result.toString()}: Job ${env.JOB_BASE_NAME} #${env.BUILD_NUMBER}",
                    body: '''${SCRIPT, template="groovy-html.template"}'''
            )
        }
    }
}