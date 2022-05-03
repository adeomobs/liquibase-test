

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
