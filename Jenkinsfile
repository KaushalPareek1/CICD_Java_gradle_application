pipeline {
  // Use any available agent
  agent any

  environment {
    // Set environment variable VERSION to Jenkins Build ID
    VERSION = "${env.BUILD_ID}"
    DOCKER_REGISTRY = "3.111.171.79:8083"  // Replace with your actual registry address
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

    stage("Docker Build & Push") {
      steps {
        script {
          // Use Docker commands to build, tag, login, push, and clean up
          withCredentials([usernamePassword(credentialsId: 'docker-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
            sh '''
              docker build -t ${DOCKER_REGISTRY}/springapp:${VERSION} .
              docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD} ${DOCKER_REGISTRY}
              docker push ${DOCKER_REGISTRY}/springapp:${VERSION}
              docker rmi ${DOCKER_REGISTRY}/springapp:${VERSION}
            '''
          }
        }
      }
    }	
  }
}
