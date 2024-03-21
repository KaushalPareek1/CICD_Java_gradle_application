pipeline {
  // Use any available agent
  agent any

  stages {
    // Sonar quality check stage
    stage("Sonar Quality Check") {
      agent {
        docker {
          // Use openjdk:11 docker image
          image 'openjdk:11'
        }
      }
      steps {
        script {
          // Configure SonarQube environment with credentials
          withSonarQubeEnv(credentialsId: 'sonar-token') {
            // Make gradlew script executable
            sh 'chmod +x gradlew'
            // Run SonarQube analysis with gradlew
            sh './gradlew sonarqube'
          }

          // Timeout after 1 hour
          timeout(time: 1, unit: 'HOURS') {
            // Wait for SonarQube analysis and get quality gate status
            def qg = waitForQualityGate()
            // Fail pipeline if quality gate is not OK
            if (qg.status != 'OK') {
              error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
          }
        }
      }
    }
  }
}
