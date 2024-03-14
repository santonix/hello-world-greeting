node('docker') {
    stage('Poll') {
        checkout scm
    }
    stage('Build & Unit test') {
        sh 'mvn clean verify -DskipITs=true'
        junit '**/target/surefire-reports/TEST-*.xml'
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
    }
    stage('Static Code Analysis') {
        withCredentials([string(credentialsId: 'jenkins-sonar-token', variable: 'SONAR_TOKEN')]) {
            sh "mvn clean verify sonar:sonar -Dsonar.projectName=hello-world-greeting -Dsonar.projectKey=hello-world-greeting -Dsonar.projectVersion=$BUILD_NUMBER -Dsonar.login=$SONAR_TOKEN"
        }
    }
    stage('Integration Test') {
        sh 'mvn clean verify -Dsurefire.skip=true'
        junit '**/target/failsafe-reports/TEST-*.xml'
        archiveArtifacts artifacts: 'target/*.war', fingerprint: true
    }
    stage('Publish') {
        script {
            def server = Artifactory.server 'default artifactory server'
            def uploadSpec = """{
                "files": [
                    {
                        "pattern": "target/hello-0.0.1.war",
                        "target": "hello-world-greeting/${BUILD_NUMBER}/",
                        "props": "Integration-Tested=Yes;Performance-Tested=No"
                    }
                ]
            }"""
            server.upload(uploadSpec)
        }
    }
    stash includes: 'target/hello-0.0.1.war,src/pt/Hello_World_Test_Plan.jmx', name: 'binary'

    stage('Run Container') {
        // Run the Docker container with the provided image
        docker.image('performance-test-agent-0.1').run('-u jenkins -v /home/jenkins:/home/jenkins')
    }

    stage('Access Container and Run Script') {
        // Set the DOCKER_HOST environment variable
        withEnv(['DOCKER_HOST=unix:///var/run/docker.sock']) {
            // Access the container as jenkins user
            docker.image('performance-test-agent-0.1').run(
                '-u jenkins -v /home/jenkins:/home/jenkins',
                 command: 'cd /home/jenkins/tomcat/bin && ./startup.sh && exec bash'
            )
        }
    }

    stage('Deploy') {
        unstash 'binary'
        sh 'cp target/hello-0.0.1.war /home/jenkins/tomcat/webapps/'
    }

    stage('Performance Testing') {
        sh '''cd /opt/jmeter/bin/
        ./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l
        $WORKSPACE/test_report.jtl'''
        step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
    }

    stage('Promote build in Artifactory') {
        withCredentials([usernameColonPassword(credentialsId: 'artifactory-account', variable: 'credentials')]) {
            sh 'curl -u${credentials} -X PUT "http://172.17.8.108:8081/artifactory/api/storage/example-project/${BUILD_NUMBER}/hello-0.0.1.war?properties=Performance-Tested=Yes"'
        }
    }
}

