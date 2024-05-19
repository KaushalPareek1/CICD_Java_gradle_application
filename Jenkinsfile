pipeline {
    agent {
        kubernetes {
            label 'jenkins-slave'
            defaultContainer 'jnlp'
            yaml """
            apiVersion: v1
            kind: Pod
            metadata:
              name: jenkins-slave
              namespace: jenkins
            spec:
              containers:
              - name: maven
                image: maven:3.8.1-openjdk-11
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/home/jenkins/agent"
                  name: "workspace-volume"
                  readOnly: false
              - name: docker
                image: docker:latest
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/home/jenkins/agent"
                  name: "workspace-volume"
                  readOnly: false
              - name: helm
                image: alpine/helm:3.5.4
                command:
                - cat
                tty: true
                volumeMounts:
                - mountPath: "/home/jenkins/agent"
                  name: "workspace-volume"
                  readOnly: false
              - name: jnlp
                image: jenkins/inbound-agent:3206.vb_15dcf73f6a_9-2
                env:
                - name: JENKINS_SECRET
                  value: "********"
                - name: JENKINS_AGENT_NAME
                  value: "jenkins-slave"
                - name: JENKINS_WEB_SOCKET
                  value: "true"
                - name: JENKINS_NAME
                  value: "jenkins-slave"
                - name: JENKINS_AGENT_WORKDIR
                  value: "/home/jenkins/agent"
                - name: JENKINS_URL
                  value: "http://13.232.197.210:8080/"
                resources:
                  requests:
                    memory: "256Mi"
                    cpu: "100m"
                volumeMounts:
                - mountPath: "/home/jenkins/agent"
                  name: "workspace-volume"
                  readOnly: false
              nodeSelector:
                kubernetes.io/os: "linux"
              restartPolicy: "Never"
              volumes:
              - emptyDir:
                  medium: ""
                name: "workspace-volume"
              dnsPolicy: "ClusterFirst"
              dnsConfig:
                nameservers:
                - "8.8.8.8"
                - "8.8.4.4"
            """
        }
    }
    stages {
        stage('Debug DNS') {
            steps {
                container('maven') {
                    sh 'nslookup github.com || dig github.com'
                }
            }
        }
    environment {
        VERSION = "${env.BUILD_ID}"
        DOCKER_REGISTRY = "my-docker-registry:5000" // Replace with your actual registry address
        KUBE_TOKEN = credentials('kubernetes-token') // Retrieving Kubernetes token from credentials
        kube_IP = "172.31.42.5"
    }
    stages {
        stage('Debug DNS') {
            steps {
                container('maven') {
                    sh 'nslookup github.com || dig github.com'
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            steps {
                container('maven') {
                    sh 'mvn clean package'
                }
            }
        }

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
                                    curl -u ${NEXUS_USER}:${NEXUS_PASSWORD} -v -X POST http://${IP}:8081/repository/helm-hosted/ --upload-file myapp-${helmVersion}.tgz
                                    rm myapp-${helmVersion}.tgz
                                """
                            }
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
            mail bcc: '', body: """
                <br>Project: ${env.JOB_NAME}
                <br>Build Number: ${env.BUILD_NUMBER}
                <br>URL: ${env.BUILD_URL}
            """, cc: '', charset: 'UTF-8', from: '', mimeType: 'text/html', replyTo: '', subject: "${currentBuild.result} CI: Project name -> ${env.JOB_NAME}", to: "your-email@example.com"
        }
    }
}

