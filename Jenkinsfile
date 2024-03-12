pipeline {
    agent { node { label 'docker' } }
    stages {
        stage('Poll') {
            steps {
                checkout scm
            }
        }
        stage('Build & Unit test') {
            steps {
                sh 'mvn clean verify -DskipITs=true'
                junit '**/target/surefire-reports/TEST-*.xml'
                archiveArtifacts 'target/*.war'
            }
        }
        stage('Build & SonarQube Analysis') {
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
            steps {
                sh 'mvn clean verify -Dsurefire.skip=true'
                junit '**/target/failsafe-reports/TEST-*.xml'
                archiveArtifacts 'target/*.war'
            }
        }
        stage('Publish') {
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
            steps {
                stash includes: 'target/hello-0.0.1.war,src/pt/Hello_World_Test_Plan.jmx', name: 'binary'
            }
        }
    



        stage('Start Tomcat') {
            steps {
                sh '''cd /home/jenkins/tomcat/bin && \
                    ./startup.sh'''
            }
        }
        stage('Deploy') {
            steps {
                unstash 'binary'
                sh 'cp target/hello-0.0.1.war /home/jenkins/tomcat/webapps/'
            }
        }
        stage('Performance Testing') {
            steps {
                sh '''cd /opt/jmeter/bin/
                ./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l $WORKSPACE/test_report.jtl'''
                step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
            }
        }
        stage('Promote Build in Artifactory') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'artifactory-account', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    script {
                        def auth = "${USERNAME}:${PASSWORD}".bytes.encodeBase64().toString()
                        sh "curl -u ${auth} -X PUT 'http://192.168.1.91:8085/api/storage/hello-world-greeting/${BUILD_NUMBER}/hello-0.0.1.war?properties=Performance-Tested=Yes'"
                    }
                }
            }
        }
    }
}
