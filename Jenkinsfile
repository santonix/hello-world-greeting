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
        stage("build & sonarqube analysis") {
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
        stage("Quality Gate") {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
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
}

pipeline {
    agent { node { label 'docker_pt' } }
    stages {
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
