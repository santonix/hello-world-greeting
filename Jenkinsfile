pipeline {
    agent {
        docker {
            image 'maven-build-slave-0.1'
            label 'docker'
            args '-v /var/run/docker.sock:/var/run/docker.sock' // Mount Docker socket for Docker-in-Docker support
        }
    }
       
    stages {
        stage('Poll') {
            steps {
                git branch: 'master', url: 'https://github.com/santonix/hello-world-greeting.git'
            }
        }

        stage('Build & Unit test') {
            steps {
                sh 'mvn clean install'
                sh 'mvn clean verify -DskipITs=true'
                junit '**/target/surefire-reports/TEST-*.xml'
                archiveArtifacts 'target/*.jar'
            }
        }

        stage('Static Code Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn clean verify sonar:sonar -Dsonar.projectName=example-project -Dsonar.projectKey=example-project -Dsonar.projectVersion=${BUILD_NUMBER}'
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
}
