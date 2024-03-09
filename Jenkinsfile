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
        stage('Static Code Analysis'){
            steps {
                script {
                  def scannerHome = tool 'sonar-scanner';
                  withSonarQubeEnv(credentialsId: 'jenkins-sonar-token') {
                    sh "${scannerHome}/bin/sonar-scanner \
                    -Dsonar.projectName=hello-world-greeting \
                    -Dsonar.projectKey=hello-world-greeting \
                    -Dsonar.java.binaries=${project.basedir}/target/classes \
                    -Dsonar.projectVersion=${BUILD_NUMBER}" 
 
                    
                  }
                }
            }
          
        }
        stage('Integration Test') {
            steps {
                sh 'mvn clean verify -Dsurefire.skip=true'
                junit '**/target/failsafe-reports/TEST-*.xml'
                archiveArtifacts 'target/*.jar'
            }
        }
        stage('Publish') {
            steps {
                script {
                    def server = Artifactory.server 'Default Artifactory Server'
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
    }
    post {
        always {
            cleanWs()
        }
    }
}

pipeline {
    agent { node { label 'docker_pt' } }
    stages {
        stage('Start Tomcat') {
            steps {
                sh '''cd /home/jenkins/tomcat/bin
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
        stage('Promote build in Artifactory') {
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

