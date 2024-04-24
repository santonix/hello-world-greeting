node('docker') {
    stage('Poll') {
      scm checkout
    }

    stage('Build & Unit test'){
       sh 'mvn clean verify -DskipITs=true';
       junit '**/target/surefire-reports/TEST-*.xml'
       archive 'target/*.jar'
    }








}
