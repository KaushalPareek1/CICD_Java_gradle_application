pipeline {
    agent any
    environment {
        VERSION = "${env.BUILD_ID}"
        DOCKER_REGISTRY = "13.201.126.141:8083"
        KUBE_TOKEN = credentials('kubernetes-token')
        kube_IP = "13.201.126.141"
    }
    stages {
        stage('Sonar Quality Check') {
            steps {
                container('maven') {
                    withSonarQubeEnv('sonar-token') {
                        sh 'mvn sonar:sonar'
                    }
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(credentialsId: 'docker-host', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                        script {
                            sh "docker build -t ${DOCKER_REGISTRY}/myapp:${VERSION} ."
                            sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD} ${DOCKER_REGISTRY}"
                            sh "docker push ${DOCKER_REGISTRY}/myapp:${VERSION}"
                            sh "docker rmi ${DOCKER_REGISTRY}/myapp:${VERSION}"
                        }
                    }
                }
            }
        }

        stage('Identifying Misconfigs Using Datree in Helm Charts') {
            steps {
                container('helm') {
                    sh 'helm datree test kubernetes/myapp/'
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

        stage('Manual Approval') {
            steps {
                script {
                    timeout(time: 10, unit: 'SECONDS') {
                        mail to: 'kaushalpareek93@gmail.com',
                            subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}",
                            body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment request <br> URL de build: ${env.BUILD_URL}",
                            mimeType: 'text/html'
                        input id: 'Deploy_Gate', message: "Deploy ${params.project_name}?", ok: 'Deploy'
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
