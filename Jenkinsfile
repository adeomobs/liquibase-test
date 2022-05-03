String manageVaultTokenId = 'my-vault-cred'
String VAULT_ADDR = 'http://127.0.0.1:8200/ui/vault/secrets'
String VAULT_PREFIX = '/ui/vault/secrets'   // No trailing slash
String VAULT_PATH = "${VAULT_PREFIX}/secret/demoTest"   

pipeline {
    agent any
    options {
        timestamps()
        buildDiscarder(logRotator(daysToKeepStr: '1095'))   // 3 years
        disableConcurrentBuilds()
    }
    stages {
        stage('echo') {
            steps {
              sh 'echo hello'
               }
        }
        
        stage('Test Vault Accesss') {
            steps {
                script{
                    withVault(configuration:[timeout: 60, vaultCredentialId: manageVaultTokenId, vaultUrl: VAULT_ADDR,], vaultSecrets: [[path: "${VAULT_PATH}", secretValues: [[envVar: 'test', vaultKey: "username"]]]]) 
                    {
                        sh 'echo $username'
                    }
                }
            }
        }

    }
}
