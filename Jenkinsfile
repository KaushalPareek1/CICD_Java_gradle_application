pipeline {
    agent any

    environment {
        VERSION = "${env.BUILD_ID}"
        IP = "65.0.104.80"
        DOCKER_REGISTRY = "${IP}:8083"
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

        stage('Identifying Misconfigs Using Datree in Helm Charts') {
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
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-host', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        dir('kubernetes/') {
                            sh '''
                                helmversion=$(helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' ')
                                tar -czvf myapp-${helmversion}.tgz myapp/
                                if docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD} ${DOCKER_REGISTRY}; then
                                    curl -u ${DOCKER_USER}:${DOCKER_PASSWORD} -v -X POST http://${IP}:8081/repository/helm-hosted/ --upload-file myapp-${helmversion}.tgz
                                    echo 'Helm chart uploaded successfully!'
                                else
                                    echo 'Docker login failed!'
                                    echo 'Helm chart upload failed!'
                                fi
                                rm myapp-${helmversion}.tgz
                            '''
                        }
                    }
                }
            }
        }

         stage('Manual Approval') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        mail bcc: '', 
                             body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", 
                             cc: '', 
                             charset: 'UTF-8', 
                             from: '', 
                             mimeType: 'text/html', 
                             replyTo: '', 
                             subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", 
                             to: "kaushalpareek93@gmail.com"
                        input(id: "DeployGate", message: "Deploy ${env.JOB_NAME}?", ok: 'Deploy')
                    }
                }
            }
        }

        stage('Deploying Application on K8s Cluster') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'jenkins-kubeconfig']) {
                        dir('kubernetes/') {
                            sh '''
                                helm upgrade --install --set image.repository="${DOCKER_REGISTRY}/springapp" --set image.tag="${VERSION}" myapp myapp/
                            '''
                        }
                    }
                }
            }
        }

        stage('Verifying App Deployment') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'jenkins-kubeconfig']) {
                        sh 'kubectl run curl --image=curlimages/curl -i --rm --restart=Never -- curl myjavaapp-myapp:8080'
                    }
                }
            }
        }
    }

    post {
        always {
            mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "kaushalpareek93@gmail.com"
        }
    }
}
