String manageVaultTokenId = 'vault-token-2'
String VAULT_ADDR = 'http://127.0.0.1:8200'
String VAULT_PATH = "/ui/vault/secrets/secret"   

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
        
    }
}
