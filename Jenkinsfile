pipeline {
    agent any
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/Shubham-Stunner/BoardGame.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('File system scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                            -Dsonar.java.binaries=. '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script { 
                    waitForQualityGate abortPipeline: false, credentialsId: 'squ_7217cd6de21c023a593032dc16626d5fb8ffc22e'  
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish Artifacts to Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Build and Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker build -t stunnershubham/boardgame:latest ."
                    }
                }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "trivy image --format table -o trivy-image-report.html stunnershubham/boardgame:latest"
                    }
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        sh "docker push stunnershubham/boardgame:latest"
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'kubeadm join 172.31.46.210:6443 --token eauwqb.v7tv9g4x5nnvl6t4 --discovery-token-ca-cert-hash sha256:5f65664950a8659d32ac865c9bb19d443bb483eae4762ffb87344757f13b1028', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.8.22:6443') {
                    sh "kubectl apply -f deployment-service.yaml"    
                    sh "kubectl get pods -n webapps"
                }
            }
        }
    }

    post {
        always {
            script {
                def jobName = env.JOB_NAME
                def buildNumber = env.BUILD_NUMBER
                def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
                def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

                def body = """
                    <html>
                    <body>
                    <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                    <h2>${jobName} - Build ${buildNumber}</h2>
                    <div style="background-color: ${bannerColor}; padding: 10px;">
                    <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                    </div>
                    <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                    </div>
                    </body>
                    </html>
                """

                emailext(
                    subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                    body: body,
                    to: 'iamshahid.shaikh4242@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}
