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

    stage('Static Code Analysis'){
        sh 'mvn clean verify sonar:sonar -Dsonar.projectName=example-project
        -Dsonar.projectKey=example-project 
        -Dsonar.projectVersion=$BUILD_NUMBER';
    }

    stage('Email Notification') {
        emailext subject: 'Jenkins Build Notification',
                  body: "Your Jenkins build ${currentBuild.result} - ${currentBuild.currentResult}",
                  recipientProviders: [[$class: 'DevelopersRecipientProvider']]
    }
}
