String manageVaultTokenId = 'vault-token-2'
String VAULT_ADDR = 'http://127.0.0.1:8200'
String VAULT_PATH = "/ui/vault/secrets/secret/demoTest"   

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
                    withVault(configuration:[timeout: 60, vaultCredentialId: 'vault-token-2', vaultUrl: VAULT_ADDR, engineVersion: 1], vaultSecrets: [[path: "${VAULT_PATH}", secretValues: [[envVar: "demoTest", vaultKey: "username"]]]]) 
                    {
                        def result = readJSON text: demoTest
                        sh 'echo result.username'
                    }
                }
            }
        }

    }
}
