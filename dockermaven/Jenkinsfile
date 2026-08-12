<![CDATA[
// Source: https://github.com/jenkinsci/pipeline-examples (MIT License)
// Test case: Docker agent and containers
pipeline {
  agent {
    docker {
      image 'maven:3.5.0-jdk-8'
      args '-v /tmp:/tmp -p 8000:8000'
    }
  }
  
  environment {
    MAVEN_OPTS = '-Xmx2048m'
  }
  
  stages {
    stage("Build") {
      steps {
        sh 'mvn --version'
        sh 'java -version'
        sh 'mvn clean compile'
      }
    }
    stage("Test") {
      steps {
        sh 'mvn test'
      }
      post {
        always {
          publishTestResults testResultsPattern: 'target/surefire-reports/*.xml'
        }
      }
    }
    stage("Package") {
      steps {
        sh 'mvn package -DskipTests'
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }
  }
}
    ]]>