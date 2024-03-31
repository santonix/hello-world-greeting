pipeline {
    agent none
    environment {
        DOCKER_CREDENTIALS = credentials('dockerhub-credentials')
    }

    stages {
        stage('Poll') {
            agent { label 'docker' }
            steps {
                checkout scm
            }
        }
        
        stage('Build & Unit test') {
            agent { label 'docker' }
            steps {
                sh 'mvn clean verify -DskipITs=true'
                junit '**/target/surefire-reports/TEST-*.xml'
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }

        stage('Build & SonarQube Analysis') {
            agent { label 'docker' }
            steps {
                withSonarQubeEnv(credentialsId: 'jenkins-sonar-token', installationName: 'sonarqube server') {
                    sh '''mvn clean verify sonar:sonar \
                        -Dsonar.projectName=hello-world-greeting \
                        -Dsonar.projectKey=hello-world-greeting \
                        -Dsonar.projectVersion=${BUILD_NUMBER} \
                        -Dsonar.java.binaries=target/classes'''
                }
            }
        }

        stage('Integration Test') {
            agent { label 'docker' }
            steps {
                sh 'mvn clean verify -Dsurefire.skip=true'
                junit '**/target/failsafe-reports/TEST-*.xml'
                archiveArtifacts artifacts: 'target/*.war', fingerprint: true
            }
        }

        stage('Publish') {
            agent { label 'docker' }
            steps {
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
        }

        stage('Stash') {
            agent { label 'docker' }
            steps {
                stash includes: 'target/hello-0.0.1.war,src/pt/Hello_World_Test_Plan.jmx', name: 'binary'
            }
        }

        stage('Start Tomcat') {
                agent {  label 'docker_pt' }
               
                   
                   
                
            
            steps {
                unstash 'binary'
                 sh './startup.sh'
               
                     
                      
                    
                 
               
            }
        }

        stage('Deploy') {
            agent { label 'docker_pt' }
            steps {
               
                sh 'cp target/hello-0.0.1.war /home/jenkins/tomcat/webapps/'
            }
        }

        stage('Performance Testing') {
            agent { label 'docker_pt' }
            steps {
                sh 'cd /opt/jmeter/bin && ./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l $WORKSPACE/test_report.jtl'
                step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
            }
        }

        stage('Promote build in Artifactory') {
            agent { label 'docker_pt' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'artifactory-account', variable: 'credentials')]) {
                    sh 'curl -u${credentials} -X PUT "http://172.17.8.108:8081/artifactory/api/storage/example-project/${BUILD_NUMBER}/hello-0.0.1.war?properties=Performance-Tested=Yes"'
                }
            }
        }
    }
     post {
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
