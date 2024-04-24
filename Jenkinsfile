node('docker') {
    stage('Poll') {
        checkout scm
    }

    stage('Build & Unit test') {
        sh 'mvn clean verify -DskipITs=true'
        junit '**/target/surefire-reports/TEST-*.xml'
    }

    stage('Archive Artifacts') {
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
    }

    stage('Email Notification') {
        success {
            emailext subject: 'Jenkins Build Notification - Success',
                      body: 'Your Jenkins build was successful.',
                      to: 'jofranco1203@gmail.com'
        }
        failure {
            emailext subject: 'Jenkins Build Notification - Failure',
                      body: 'Your Jenkins build failed.',
                      to: 'jofranco1203@gmail.com'
        }
    }
}
