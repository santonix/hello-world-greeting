node('docker') {
    stage('Poll') {
        checkout scm
    }

    stage('Build & Unit test') {
        sh 'mvn clean verify -DskipITs=true'
        junit '**/target/surefire-reports/TEST-*.xml'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
    }

    post {
        success {
            // Add success post-build actions here
            emailext subject: 'Jenkins Build Notification - Success',
                      body: 'Your Jenkins build was successful.',
                      to: 'jofranco1203@gmail.com'
        }
        failure {
            // Add failure post-build actions here
            emailext subject: 'Jenkins Build Notification - Failure',
                      body: 'Your Jenkins build failed.',
                      to: 'jofranco1203@gmail.com'
        }
    }
}
