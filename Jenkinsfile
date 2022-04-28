/*
 *=================================================================================================================
 *NAME: Jenkinsfile
 *DESCRIPTION: Jenkinsfile
 *=================================================================================================================
 */
library identifier: 'lib@master', retriever: modernSCM(
[$class : 'GitSCMSource', remote: 'git@github.com:rgare/rga-deployment.git',
credentialsId: 'jenkins-autokeygen-rga-deployment', includes: '*'])

String nodeLabel = 'wf-docker'

// TO USE JENKINS CREDENTIALS TO STORE YOUR IAM CREDENTIALS, USE THIS BLOCK OF CODE.
// THIS HAS PRIMARILY BEEN IMPLEMENTED TO SIMPLIFY THIS EXAMPLE AND SHOULD NOT BE
// THE LOCATION OF YOUR CREDENTIALS
def awsCredentialsId = 'aws4ccs-asap-global-ci-deploy' // This will be the identifier in the Jenkins Credentials
def specificCause = currentBuild.getBuildCauses('hudson.model.Cause$UserIdCause')
def jenkinsUID = "${specificCause.userId[0]}"
def aws_env_stage_map = [
    "poc": "poc",
    "play": "poc",
    "dev": "sdlc",
    "test": "sdlc",
    "prod": "cust"
]

def stage_aws_account_number_map = [
    "poc": "462374163868",
    "sdlc": "983735142482",
    "cust": "372519005580"
]
def log_choices = ['ERROR','WARN','INFO','DEBUG','TRACE']
def assume_role_duration = 3600
def region = 'us-east-1'  // default region
def env_choices = env.JENKINS_URL.contains('https://ccs-prod-ci.desapp-st1.rgare.net') ? ['prod'] : ['poc', 'play','dev', 'test']

pipeline {
    agent { label nodeLabel }
    options {
        timestamps()
        buildDiscarder(logRotator(daysToKeepStr: '1095'))   // 3 years
        disableConcurrentBuilds()
    }
    // triggers {
    //     githubPush()
    // }
    parameters {
        choice( choices: ['apply', 'plan'], name: 'TF_ACTION' )
        choice( choices: env_choices,description:'AWS Environment to Deploy', name: 'APP_ENV' )
        string( defaultValue: "release", description: "Build Branch", name: 'PLATFORM_UI_BUILD_BRANCH' )
        string( defaultValue: "", description: "Artifact Version to Deploy", name: 'PLATFORM_UI_VERSION' )
        string( defaultValue: "release", description: "Build Branch", name: 'USER_API_BUILD_BRANCH' )
        string( defaultValue: "", description: "Artifact Version to Deploy", name: 'USER_API_VERSION' )
        choice( choices: log_choices, description: 'Terraform Logging Level', name: 'TF_LOG_LEVEL')
        string( defaultValue: "-input=false -no-color", description: 'Options for Terraform CLI', name: 'TF_OPTS')

    }
    environment {

        TF_IN_AUTOMATION      = '1'
        DEPLOY_ROLE = "${APP_ENV in ['poc','play'] ? 'amy_test_role' : 'ccs-asap-global-ci-deploy'}"
        TF_LOG = "${TF_LOG_LEVEL}"

    }
    stages {
        stage('Preparation') {
            steps {
                script {
                    //Clean the workspae prior to execution.
                    cleanWs()
                    currentBuild.description = "$APP_ENV"
                    //Explicitly checkout the code after cleaning the workspace
                    checkout scm
                    env.STAGE = aws_env_stage_map["$APP_ENV"]
                    env.AWS_ACCOUNT_NUMBER = stage_aws_account_number_map["$env.STAGE"]
                    env.UI_DEPLOY_BUCKET = ""
                    env.DEPLOY_USER_API = ""
                }
            }
        }
        stage('Download Zip'){
            when {
                expression { params.PLATFORM_UI_VERSION != '' || params.USER_API_VERSION != '' }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'css-fusion-nexus',
                    usernameVariable: 'NEXUS_USERNAME',
                    passwordVariable: 'NEXUS_PASSWORD'
                )]) {
                    script {
                        if (PLATFORM_UI_VERSION != "") {
                            def appName = 'fusion_ui'
                            currentBuild.description += ", ${appName}: ${PLATFORM_UI_VERSION}"
                            env.UI_DEPLOY_BUCKET = "-target=module.ui-platform"
                            // TODO: refactor below to reusable method
                            NEXUS_PREFIX = "https://nexus.rgare.net/repository/ccs-raw/${appName}"
                            env.ZIP_ARTIFACT_FILENAME = "${appName}-${PLATFORM_UI_VERSION}.zip"
                            sh """
                                wget -nv ${NEXUS_PREFIX}/${PLATFORM_UI_BUILD_BRANCH}/${PLATFORM_UI_VERSION}/${env.ZIP_ARTIFACT_FILENAME}
                            """
                            RgaZipUtils.expandArchive(this, "${appName}/", "${env.ZIP_ARTIFACT_FILENAME}")
                        }
                        if (USER_API_VERSION != "") {
                            def appName = 'fusion_api_users'
                            currentBuild.description += ", ${appName}: ${USER_API_VERSION}"
                            env.DEPLOY_USER_API  = "-target=module.api-user"
                            // TODO: refactor below to reusable method
                            NEXUS_PREFIX = "https://nexus.rgare.net/repository/ccs-raw/${appName}"
                            env.ZIP_ARTIFACT_FILENAME = "${appName}-${USER_API_VERSION}.zip"
                            sh """
                                wget -nv ${NEXUS_PREFIX}/${USER_API_BUILD_BRANCH}/${USER_API_VERSION}/${env.ZIP_ARTIFACT_FILENAME}
                            """
                            RgaZipUtils.expandArchive(this, "${appName}/", "${env.ZIP_ARTIFACT_FILENAME}")
                        }
                    }

                }
            }
        }

        stage('RunTerraform') {
            steps {
                withAWS(region:"${region}", credentials:"${awsCredentialsId}-${env.STAGE}", role:"${env.DEPLOY_ROLE}", roleAccount:"${env.AWS_ACCOUNT_NUMBER}", duration:"${assume_role_duration}", roleSessionName: 'fusion-infra-session'){
                    sh script: """
                        set -Eeuxo pipefail
                        cd terraform
                        tfenv use min-required || (export TFENV_CURL_OUTPUT=0 && tfenv install min-required && tfenv use min-required)
                        chmod +x ../jenkins/${APP_ENV}.sh
                        ../jenkins/${APP_ENV}.sh ${TF_ACTION} ${TF_OPTS} ${UI_DEPLOY_BUCKET} ${DEPLOY_USER_API}
                    """
                }
            }
        }
        stage('Trigger E2E') {
            when {
                expression { return !['poc', 'play'].contains(params.APP_ENV) }
            }
            steps {
                script {
                    if (PLATFORM_UI_VERSION != "") {
                        build job: 'Test_E2E_Platform_UI', parameters: [
                        [$class: 'StringParameterValue', name: 'ENVIRONMENT', value: params.APP_ENV],
                        [$class: 'StringParameterValue', name: 'BROWSER_CONFIG', value: 'chrome'],
                        ],
                        quietPeriod: 0,
                        wait: false
                    }
                }
            }
        }
    }


    post {
        cleanup {
            cleanWs(cleanWhenAborted: false)
        }
    }
}
