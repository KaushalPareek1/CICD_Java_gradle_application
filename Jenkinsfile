pipeline {
  agent any

  environment {
    VERSION = "${env.BUILD_ID}"
    IP = "13.126.5.61"
    DOCKER_REGISTRY = "${IP}:8083"  // Replace with your actual registry address
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
            sh 'helm datree test myapp/'
          }
        }
      }
    }
    
    stage('pushing the helm charts to nexus'){
      steps{
        script{
          withCredentials([usernamePassword(credentialsId: 'docker-host', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
            dir('kubernetes/') {
             sh '''
              helmversion=$( helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
              tar -czvf myapp-${helmversion}.tgz myapp/
              # Check for successful login before pushing
              if docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD} ${DOCKER_REGISTRY}; then
              curl -u ${DOCKER_USER}:${DOCKER_PASSWORD} -v -X POST http://${IP}:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz
              echo 'Helm chart uploaded successfully!'
              else
              echo 'Docker login failed!'
              echo 'Helm chart upload failed!'
              # Handle upload failure here (e.g., error notification)
              fi
              rm myapp-${helmversion}.tgz
              '''

            }
          }
        }
      }
    }
  
  }
  
  post {
    always {
      mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "kaushalpareek93@gmail.com";  
    }
  }
}

