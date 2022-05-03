 *===========================================
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
// TO USE VAULT TO STORE YOUR IAM CREDENTIALS (RECOMMENDED), USE THIS BLOCK OF CODE.
// -------- ALSO SEE SECTION BELOW MARKED WITH 'DANGER WILL ROBINSON' --------------
String manageVaultTokenId = 'ccs-aws-vault-reader'
String VAULT_ADDR = 'https://serviceregistry.rgare.net:8201'
String VAULT_PREFIX = '/ui/vault/secrets'   // No trailing slash
String VAULT_PATH = "${VAULT_PREFIX}/secret/list/ccsaws/nonprod"   // No trailing slash
String TERRAFORM_CREDS = 'myapp/terraform-ci-deploy-sdlc'
String DB_USERNAME_PREFIX = 'LIQUIBASE_COMMAND_USERNAME'
String DB_PASSWORD_PREFIX = 'LIQUIBASE_COMMAND_PASSWORD'

// Vault Plugin Configuration
def vaultCredentials
def vaultConfiguration = [
         vaultUrl: VAULT_ADDR,
         vaultCredentialId: manageVaultTokenId,
         engineVersion: 1,
 ]

// // Vault Path-to-Variable Mapping
 def vaultSecrets = [
         [
                 path: "${VAULT_PATH}/${TERRAFORM_CREDS}",   // ITAM shared
                 secretValues: [
                         [envVar: 'LIQUIBASE_COMMAND_USERNAME', vaultKey: 'aws_access_key_id'],
                         [envVar: 'LIQUIBASE_COMMAND_PASSWORD', vaultKey: 'aws_secret_access_key'],
                 ],
         ],
 ]

//-------------------------------------new code-----------------------------------------------------------------------------
def liquibase_creds = [
    "poc": "ccs_clientpath_poc_db_creds",
    "play": "ccs_clientpath_play_db_creds",
    "dev": "ccs_clientpath_dev_db_creds",
    "test": "ccs_clientpath_test_db_creds",
    "prod": "ccs_clientpath_prod_db_creds",
]
def user_db_urls = [
    "poc": "jdbc:postgresql://poc-temp-test-db.cluster-cmivzoefmwyf.us-east-1.rds.amazonaws.com/schuser?sslmode=require",
    "play": "jdbc:postgresql://poc-temp-test-db.cluster-cmivzoefmwyf.us-east-1.rds.amazonaws.com/schuser?sslmode=require",
    "dev": "jdbc:postgresql://rdsdasappg-cluster.cluster-cdymkminubdi.us-east-1.rds.amazonaws.com/schuser?sslmode=require",
    "test": "jdbc:postgresql://rdstasappg-cluster.cluster-cdymkminubdi.us-east-1.rds.amazonaws.com/schuser?sslmode=require",
   "prod": "jdbc:postgresql://",
]

def case_db_urls = [
    "poc": "jdbc:postgresql://poc-temp-test-db.cluster-cmivzoefmwyf.us-east-1.rds.amazonaws.com/schcase?sslmode=require",
    "play": "jdbc:postgresql://poc-temp-test-db.cluster-cmivzoefmwyf.us-east-1.rds.amazonaws.com/schcase?sslmode=require",
    "dev": "jdbc:postgresql://rdsdasappg-cluster.cluster-cdymkminubdi.us-east-1.rds.amazonaws.com/schcase?sslmode=require",
    "test": "jdbc:postgresql://rdstasappg-cluster.cluster-cdymkminubdi.us-east-1.rds.amazonaws.com/schcase?sslmode=require",
   "prod": "jdbc:postgresql://",
]
//------------------------------------------end----------------------------------------------------------------------------------#
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
        string( defaultValue: "release", description: "Build Branch", name: 'ASAP_UI_BUILD_BRANCH' )
        string( defaultValue: "", description: "Artifact Version to Deploy", name: 'ASAP_UI_VERSION' )
        string( defaultValue: "release", description: "Build Branch", name: 'USER_API_BUILD_BRANCH' )
        string( defaultValue: "", description: "Artifact Version to Deploy", name: 'USER_API_VERSION' )
        string( defaultValue: "release", description: "Build Branch", name: 'CASE_API_BUILD_BRANCH' )
        string( defaultValue: "", description: "Artifact Version to Deploy", name: 'CASE_API_VERSION' )
        choice( choices: log_choices, description: 'Terraform Logging Level', name: 'TF_LOG_LEVEL')
        string( defaultValue: "-input=false -no-color", description: 'Options for Terraform CLI', name: 'TF_OPTS')
        string(name: 'VAULT_PATH', defaultValue: 'v1/secret/ccsaws/nonprod/', description: 'Provide the vault path')

    }
    environment {

        TF_IN_AUTOMATION      = '1'
        DEPLOY_ROLE = "${APP_ENV in ['poc','play'] ? 'amy_test_role' : 'ccs-asap-global-ci-deploy'}"
        TF_LOG = "${TF_LOG_LEVEL}"

    }
    stages {
        stage('Test Vault Accesss') {
            steps {
                withCredentials([usernamePassword(
                            credentialsId: 'ccs-aws-vault-reader',
                            usernameVariable: 'VAULT_USERNAME',
                            passwordVariable: 'VAULT_PASSWORD'
                        )]) {
                            script {
                                echo 'hello this is about to get a token'
                          // Get a token
                                def vaultToken = RgaVaultLdap.token(this, env.VAULT_USERNAME, env.VAULT_PASSWORD)
                                echo "${vaultToken}"
                                // Read individual secrets
                                def secret = RgaVaultLdap.secret(this, vaultToken, '/v1/secret/ccsaws/nonprod/delete-me', 'value')
                                echo "${secret}"
                                def collection = RgaVaultLdap.listPaths(this, vaultToken, '/v1/secret/ccsaws/nonprod')
                                echo "${collection}"

               }
           }
        
        }
    }
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
                    env.UI_DEPLOY = ""
                    env.API_DEPLOY = ""
                    //the sh script below downloads the liquibase cli commands and allows us to run liquibase on this jenkins job 
                    sh 'wget --no-verbose -c https://github.com/liquibase/liquibase/releases/download/v4.8.0/liquibase-4.8.0.tar.gz -O - | tar -xz'
                }
            }
        }
        stage('Download Zip'){
            when {
                expression { params.PLATFORM_UI_VERSION != '' || params.ASAP_UI_VERSION != '' || params.USER_API_VERSION != '' || params.CASE_API_VERSION != '' }
            }
            steps {
          withCredentials([usernamePassword(
                    credentialsId: 'ccs-clientpath-nexus',
                    usernameVariable: 'NEXUS_USERNAME',
                    passwordVariable: 'NEXUS_PASSWORD'
                )]) {
                    script {
                        if (PLATFORM_UI_VERSION != "") {
                            def appName = 'platform_ui'
                            currentBuild.description += ", ${appName}: ${PLATFORM_UI_VERSION}"
                            env.UI_DEPLOY = "-var=deploy_platform_ui=true "
                            // TODO: refactor below to reusable method
                            NEXUS_PREFIX = "https://nexus.rgare.net/repository/ccs-raw/${appName}"
                            env.ZIP_ARTIFACT_FILENAME = "${appName}-${PLATFORM_UI_VERSION}.zip"
                            sh """
                                wget -nv ${NEXUS_PREFIX}/${PLATFORM_UI_BUILD_BRANCH}/${PLATFORM_UI_VERSION}/${env.ZIP_ARTIFACT_FILENAME}
                            """
                            RgaZipUtils.expandArchive(this, "${appName}/", "${env.ZIP_ARTIFACT_FILENAME}")
                        }
                        if (ASAP_UI_VERSION != "") {
                            def appName = 'asap_ui'
                            currentBuild.description += ", ${appName}: ${ASAP_UI_VERSION}"
                            env.UI_DEPLOY += "-var=deploy_asap_ui=true "
                            // TODO: refactor below to reusable method
                            NEXUS_PREFIX = "https://nexus.rgare.net/repository/ccs-raw/${appName}"
                            env.ZIP_ARTIFACT_FILENAME = "${appName}-${ASAP_UI_VERSION}.zip"
                            sh """
                                wget -nv ${NEXUS_PREFIX}/${ASAP_UI_BUILD_BRANCH}/${ASAP_UI_VERSION}/${env.ZIP_ARTIFACT_FILENAME}
                            """
                            RgaZipUtils.expandArchive(this, "${appName}/", "${env.ZIP_ARTIFACT_FILENAME}")
                        }
                        if (USER_API_VERSION != "") {
                            def appName = 'api_users'
                            currentBuild.description += ", ${appName}: ${USER_API_VERSION}"
                            env.API_DEPLOY  = "-var=deploy_user_api=true "
                            // TODO: refactor below to reusable method
                            NEXUS_PREFIX = "https://nexus.rgare.net/repository/ccs-raw/${appName}"
                            env.ZIP_ARTIFACT_FILENAME = "${appName}-${USER_API_VERSION}.zip"
                            sh """
                                wget -nv ${NEXUS_PREFIX}/${USER_API_BUILD_BRANCH}/${USER_API_VERSION}/${env.ZIP_ARTIFACT_FILENAME}
                            """
                            RgaZipUtils.expandArchive(this, "${appName}/", "${env.ZIP_ARTIFACT_FILENAME}")
                             sh """
                            withVault([
                                    configuration: vaultConfiguration,
                                    vaultSecrets: vaultSecrets,
                            ]) {
                            }
                            ls -la
                            echo 'liquibase'
                            ls liquibase                            
                            ./liquibase --log-level=SEVERE status --verbose --changelog-file=api_users/database/main-changelog.xml --url=${user_db_urls[APP_ENV]} --default-schema-name=sch_user
                            ./liquibase update --changelog-file=api_users/database/main-changelog.xml --url=${user_db_urls[APP_ENV]} --default-schema-name=sch_user 
                            """
                        }
                        if (CASE_API_VERSION != "") {
                            def appName = 'api_cases'
                            currentBuild.description += ", ${appName}: ${CASE_API_VERSION}"
                            env.API_DEPLOY  += "-var=deploy_case_api=true "
                            // TODO: refactor below to reusable method
                            NEXUS_PREFIX = "https://nexus.rgare.net/repository/ccs-raw/${appName}"
                            env.ZIP_ARTIFACT_FILENAME = "${appName}-${CASE_API_VERSION}.zip"
                            sh """
                                wget -nv ${NEXUS_PREFIX}/${CASE_API_BUILD_BRANCH}/${CASE_API_VERSION}/${env.ZIP_ARTIFACT_FILENAME}
                            """
                            RgaZipUtils.expandArchive(this, "${appName}/", "${env.ZIP_ARTIFACT_FILENAME}")
                            // Deploy DB
                            sh """
                            withVault([
                                    configuration: vaultConfiguration,
                                    vaultSecrets: vaultSecrets,
                            ]) {
                            }
                            ls -la
                            echo 'liquibase'
                            ls liquibase                             
                            ./liquibase --log-level=SEVERE status --verbose --changelog-file=api_cases/database/main-changelog.xml --url=${case_db_urls[APP_ENV]} --default-schema-name=sch_case
                            ./liquibase update --changelog-file=api_cases/database/main-changelog.xml --url=${case_db_urls[APP_ENV]} --default-schema-name=sch_case --username= 
                            """
                        }
                    }

                }
            }
        }
       /*   stage('Run Liquibase') {// so this stage downloands the liquibase cli from the link and allows liquibase to run the liquibase status commands and liquibase update command
            steps {             // but now the want me to run lines 203 in the preparation stage and add the 204-211 to the user api reference and case api refenrece above, 
                    script {
                         Environment variables
                         LIQUIBASE_COMMAND_USERNAME
                         LIQUIBASE_COMMAND_PASSWORD
                        withVault([
                                 configuration: vaultConfiguration,
                                 vaultSecrets: vaultSecrets,
                         ]) {
                         }
                         Liquibase binary
                         https://github.com/liquibase/liquibase/releases/download/v4.8.0/liquibase-4.8.0.tar.gz
                         # liquibase update --changelog-file=${liquibase_changelog-file[APP_ENV]} --url=${case_db_urls[APP_ENV]} --username=${LIQUIBASE_USER} --password=${LIQUIBASE_PWD}
                        sh """
                        wget --no-verbose -c https://github.com/liquibase/liquibase/releases/download/v4.8.0/liquibase-4.8.0.tar.gz -O - | tar -xz
                        ls -la
                        ls liquibase
                        ./liquibase --log-level=FINE status --verbose --changelog-file=api_cases/database/main-changelog.xml --url=${case_db_urls[APP_ENV]} --default-schema-name=sch_case --username=LIQUIBASE_COMMAND_USERNAME --password=LIQUIBASE_COMMAND_PASSWORD
                        ./liquibase update --changelog-file=api_cases/database/main-changelog.xml --url=${case_db_urls[APP_ENV]} --default-schema-name=sch_case --username=LIQUIBASE_COMMAND_USERNAME --password=LIQUIBASE_COMMAND_PASSWORD
                        """
                }
           }
        }*/
        // stage('RunTerraform') {
        //     steps {
        //         withAWS(region:"${region}", credentials:"${awsCredentialsId}-${env.STAGE}", role:"${env.DEPLOY_ROLE}", roleAccount:"${env.AWS_ACCOUNT_NUMBER}", duration:"${assume_role_duration}", roleSessionName: 'clientpath-infra-session'){
        //             sh script: """
        //                 set -Eeuxo pipefail
        //                 cd terraform
        //                 tfenv use min-required || (export TFENV_CURL_OUTPUT=0 && tfenv install min-required && tfenv use min-required)
        //                 chmod +x ../jenkins/${APP_ENV}.sh
        //                 ../jenkins/${APP_ENV}.sh ${TF_ACTION} ${TF_OPTS} ${UI_DEPLOY} ${API_DEPLOY}
        //             """
        //         }
        //     }
        // }
        // stage('Trigger E2E') {
        //     when {
        //         expression { return !['poc', 'play'].contains(params.APP_ENV) }
        //     }
        //     steps {
        //         script {
        //             if (PLATFORM_UI_VERSION != "") {
        //                 build job: 'Test_E2E_Platform_UI', parameters: [
        //                 [$class: 'StringParameterValue', name: 'ENVIRONMENT', value: params.APP_ENV],
        //                 [$class: 'StringParameterValue', name: 'BROWSER_CONFIG', value: 'chrome'],
        //                 ],
        //                 quietPeriod: 0,
        //                 wait: false
        //             }
        //         }
        //     }
        // }

    }
       post {
        cleanup {
            cleanWs(cleanWhenAborted: false)
        }
    }

}
/*def liquibaseUpdate() {
    // do stuff
    // sh ''
}*/


