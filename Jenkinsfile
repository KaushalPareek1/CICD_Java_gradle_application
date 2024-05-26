pipeline {
    agent any

    environment {
        VERSION = "${env.BUILD_ID}"
        DOCKER_REGISTRY = "13.201.126.141:8083"
        KUBE_TOKEN = credentials('kubernetes-token')
        kube_IP = "13.201.126.141"
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
 
        stage('Pushing the Helm Charts to Nexus') {
            steps {
                container('helm') {
                    withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASSWORD')]) {
                        script {
                            def helmVersion = sh(script: "helm show chart kubernetes/myapp | grep version | cut -d: -f 2 | tr -d ' '", returnStdout: true).trim()
                            sh """
                                tar -czvf myapp-${helmVersion}.tgz kubernetes/myapp/
                                curl -u ${NEXUS_USER}:${NEXUS_PASSWORD} -v -X POST http://${kube_IP}:8081/repository/helm-hosted/ --upload-file myapp-${helmVersion}.tgz
                                rm myapp-${helmVersion}.tgz
                            """
                        }
                    }
                }
            }
        }
     
     stage('Deploying Application on K8s Cluster') {
            steps {
                container('helm') {
                    withKubeConfig([credentialsId: 'kubernetes-token', serverUrl: "https://${kube_IP}:6443"]) {
                        dir('kubernetes/') {
                            sh '''
                                helm upgrade --install --set image.repository="${DOCKER_REGISTRY}/myapp" --set image.tag="${VERSION}" myapp myapp/
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            mail to: "your-email@example.com",
                subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}",
                body: """
                    <br>Project: ${env.JOB_NAME}
                    <br>Build Number: ${env.BUILD_NUMBER}
                    <br>URL: ${env.BUILD_URL}
                """,
                mimeType: 'text/html'
        }
    }
}
