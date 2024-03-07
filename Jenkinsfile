pipeline {
    agent {
        node { 
            label 'docker'
        }
    }
    
    stages {
        stage('Poll') {
            steps {
                script {
                    checkout scm
                }
            }
        }
        
        stage('Build & Unit test') {
            steps {
                script {
                    sh 'mvn clean verify -DskipITs=true'
                    junit '**/target/surefire-reports/TEST-*.xml'
                    archive 'target/*.jar'
                }
            }
        }
        
        stage('Static Code Analysis') {
            steps {
                withCredentials([string(credentialsId: 'jenkins-sonar-token', variable: 'SONAR_TOKEN')]) {
                    script {
                        sh "mvn clean verify sonar:sonar -Dsonar.projectName=hello-world-greeting -Dsonar.projectKey=hello-world-greeting -Dsonar.projectVersion=$BUILD_NUMBER -Dsonar.login=${env.SONAR_TOKEN}"
                    }
                }
            }
        }
        
        stage('Integration Test') {
            steps {
                script {
                    sh 'mvn clean verify -Dsurefire.skip=true'
                    junit '**/target/failsafe-reports/TEST-*.xml'
                    archive 'target/*.jar'
                }
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
                                "target": "helloworld-greeting-project/${BUILD_NUMBER}/",
                                "props": "Integration-Tested=Yes;Performance-Tested=No"
                            }
                        ]
                    }"""
                    server.upload(uploadSpec)
                }
            }
        }
    }
    
    // Add post section if needed
}
