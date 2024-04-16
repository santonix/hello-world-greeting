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
                // Modify Docker run command to mount Maven repository
                script {
                    docker.image('maven:latest').inside("-u 131:139 -v /home/bonny/.m2/repository:/root/.m2/repository") {
                        withSonarQubeEnv(credentialsId: 'sonar-token', installationName: 'sonarqube server') {
                            sh '''mvn clean verify sonar:sonar \
                                -Dsonar.projectName=hello-world-greeting \
                                -Dsonar.projectKey=hello-world-greeting \
                                -Dsonar.projectVersion=${BUILD_NUMBER} \
                                -Dsonar.java.binaries=target/classes'''
                        }
                    }
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
            agent {
                label 'docker_pt'
            }
            steps {
                // Run the Docker container with the provided image
                docker.image('santonix/santonix/performance-test-agent-0.1').inside("-u jenkins -v /home/jenkins:/home/jenkins") {
                    // Inside the container
                    // Change directory to /home/jenkins/tomcat/bin and run startup.sh
                    sh 'cd /home/jenkins/tomcat/bin && ./startup.sh'
                }
            }
        }

        stage('Deploy') {
            steps {
                // Retrieve stashed artifacts and deploy
                unstash 'binary'
                sh 'cp target/hello-0.0.1.war /home/jenkins/tomcat/webapps/'
            }
        }

        stage('Performance Testing') {
            steps {
                // Execute JMeter for performance testing
                sh '''
                    #!/bin/bash -l
                    cd /opt/jmeter/bin/
                    ./jmeter.sh -n -t $WORKSPACE/src/pt/Hello_World_Test_Plan.jmx -l $WORKSPACE/test_report.jtl
                '''
                // Archive JMeter test results
                step([$class: 'ArtifactArchiver', artifacts: '**/*.jtl'])
            }
        }

        stage('Promote build in Artifactory') {
            steps {
                // Promote build in Artifactory with credentials
                withCredentials([usernameColonPassword(credentialsId: 'artifactory-account', variable: 'credentials')]) {
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
