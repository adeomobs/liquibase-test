 //===========================================
 //NAME: Jenkinsfile
 //DESCRIPTION: Jenkinsfile
 //=================================================================================================================
 //*/
// TO USE VAULT TO STORE YOUR IAM CREDENTIALS (RECOMMENDED), USE THIS BLOCK OF CODE.
// -------- ALSO SEE SECTION BELOW MARKED WITH 'DANGER WILL ROBINSON' --------------
String manageVaultTokenId = 'ccs-aws-vault-reader'
String VAULT_ADDR = 'http://127.0.0.1:8200'
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
    agent any
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

    }
       post {
        cleanup {
            cleanWs(cleanWhenAborted: false)
        }
    }

}
