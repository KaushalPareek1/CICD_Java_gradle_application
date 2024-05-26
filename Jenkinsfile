pipeline {
    agent any
    environment {
        VERSION = "${env.BUILD_ID}"
        DOCKER_REGISTRY = "13.201.126.141:8083" // Replace with your actual registry address
        KUBE_TOKEN = credentials('kubernetes-token') // Retrieving Kubernetes token from credentials
        kube_IP = "13.201.126.141"
    }
    stages {
        stage('Sonar Quality Check') {
            steps {
                container('maven') {
                    script {
                        withSonarQubeEnv('sonar-token') {
                            sh 'mvn sonar:sonar'
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
        }

        stage('Docker Build & Push') {
            steps {
                container('docker') {
                    script {
                        withCredentials([usernamePassword(credentialsId: 'docker-host', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                            sh '''
                                docker build -t ${DOCKER_REGISTRY}/myapp:${VERSION} .
                                docker login -u ${DOCKER_USER} -p ${DOCKER_PASSWORD} ${DOCKER_REGISTRY}
                                docker push ${DOCKER_REGISTRY}/myapp:${VERSION}
                                docker rmi ${DOCKER_REGISTRY}/myapp:${VERSION}
                            '''
                        }
                    }
                }
            }
        }

        stage('Identifying Misconfigs Using Datree in Helm Charts') {
            steps {
                container('helm') {
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
                        dir('kubernetes/') {
                            script {
                                def helmVersion = sh(script: "helm show chart myapp | grep version | cut -d: -f 2 | tr -d ' '", returnStdout: true).trim()
                                sh """
                                    tar -czvf myapp-${helmVersion}.tgz myapp/
                                    curl -u ${NEXUS_USER}:${NEXUS_PASSWORD} -v -X POST http://${kube_IP}:8081/repository/helm-hosted/ --upload-file myapp-${helmVersion}.tgz
                                    rm myapp-${helmVersion}.tgz
                                """
                            }
                        }
                    }
                }
            }
        }
        stage('manual approval'){
            steps{
                script{
                    timeout(10) {
                        mail bcc: '', body: "<br>Project: ${env.JOB_NAME} <br>Build Number: ${env.BUILD_NUMBER} <br> Go to build url and approve the deployment request <br> URL de build: ${env.BUILD_URL}", cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "kaushalpareek93@gmail.com";  
                        input(id: "Deploy Gate", message: "Deploy ${params.project_name}?", ok: 'Deploy')
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
            mail bcc: '', body: """
                <br>Project: ${env.JOB_NAME}
                <br>Build Number: ${env.BUILD_NUMBER}
                <br>URL: ${env.BUILD_URL}
            """, cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "your-email@example.com"
        }
    }
}
