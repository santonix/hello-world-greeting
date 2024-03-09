node('docker') {
    stage('Poll') {
        checkout scm
    }
    stage('Build & Unit test') {
        sh 'mvn clean verify -DskipITs=true'
        junit '**/target/surefire-reports/TEST-*.xml'
        archiveArtifacts 'target/*.war'
    }
    
    stage('Static Code Analysis') {
        withCredentials([string(credentialsId: 'jenkins-sonar-token', variable: 'SONAR_TOKEN')]) {
            sh "mvn clean verify sonar:sonar -Dsonar.projectName=hello-world-greeting -Dsonar.projectKey=hello-world-greeting -Dsonar.projectVersion=$BUILD_NUMBER -Dsonar.login=${env.SONAR_TOKEN}"
        }
    }
    
    stage('Publish') {
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
    stash includes: 
   'target/hello-0.0.1.war,src/pt/Hello_World_Test_Plan.jmx',      
    name: 'binary'    
}

node('docker_pt') {
    stage('Start Tomcat') {
       sh '/home/jenkins/tomcat/bin/startup.sh'
        
    }
    stage('Deploy') {
        unstash 'binary'
        sh 'cp target/hello-0.0.1.war /home/jenkins/tomcat/webapps/'
    }
    stage('Performance Testing') {
        sh '''cd /opt/jmeter/bin/
        ./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l $WORKSPACE/test_report.jtl'''
        step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
    }
    stage('Promote build in Artifactory') {
        withCredentials([usernameColonPassword(credentialsId: 'artifactory-account', variable: 'credentials')]) {
            sh 'curl -u${credentials} -X PUT "http://192.168.1.91:8085/api/storage/hello-world-greeting/${BUILD_NUMBER}/hello-0.0.1.war?properties=Performance-Tested=Yes"'
        }
    }
}
