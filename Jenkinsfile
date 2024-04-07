pipeline {
  agent any

  environment {
    VERSION = "${env.BUILD_ID}"
    DOCKER_REGISTRY = "13.201.31.22:8083"  // Replace with your actual registry address
  }

  stages {
    stage("Sonar Quality Check") {
      agent {
        docker { image 'openjdk:11' }
      }
      steps {
        script {
          withSonarQubeEnv(credentialsId: 'sonar-token') {
            sh 'chmod +x gradlew'
            sh './gradlew sonarqube'
          }
          timeout(time: 1, unit: 'HOURS') {
            def qg = waitForQualityGate()
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
          withCredentials([usernamePassword(credentialsId: 'docker-host', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
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
     stage('Identifying Misconfigs Using Datree in Helm Charts') { // Corrected stage name
      steps {
        script {
          dir('kubernetes/') {
            sh 'helm datree test myapp/ --set datree.offline=true --set datree.local=true'
          }
        }
      }
    } 
  }
}

