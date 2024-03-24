pipeline {
  // Use any available agent
  agent any

  environment {
    // Set environment variable VERSION to Jenkins Build ID
    VERSION = "${env.BUILD_ID}"
  }

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

    // Build and push Docker image stage
    stage("Docker Build & Push") {
        steps {
        script {
          // Use Docker commands to build, tag, login, push, and clean up
          withCredentials([string(credentialsId: 'docker_pass', variable: 'docker_password')]) {
            sh '''
              docker build -t 13.201.6.4:8083/springapp:${VERSION} .
              docker login -u admin --password-stdin $docker_pass 
              docker push 13.201.6.4:8083/springapp:${VERSION}
              docker rmi 13.201.6.4:8083/springapp:${VERSION}
            '''
          }
        }
      }
    }
  }
}
