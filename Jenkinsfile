pipeline {
    agent any
    
    tools {
        maven "maven" 
        jdk "jdk"     
    }
    
    environment {
        SCANNER_HOME = tool 'sonar'
    }

    stages {
        stage('Git Checkout') {
            steps {
                script {
                    git branch: 'main', url: 'https://github.com/tulshiramKIC/BoardGame.git'
                }
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Test') {
            steps {
                sh "mvn test -DskipTests=true"
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    // Run SonarQube analysis using the configured SonarScanner
                    withSonarQubeEnv('sonar') {
                        sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame -Dsonar.java.binaries=. '''
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package -DskipTests=true"
            }
        }
 
        stage('Push To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: '25339dd1-7232-4a6d-bb89-fe883af3bdd8', jdk: 'jdk', maven: 'maven', mavenSettingsConfig: '', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Docker Build and Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build -t tdk785/boardgame:latest .'
                    }   
                }
            }
        }
        
        stage('Trivy image Scan') {
            steps {
                sh "trivy image --format table -o trivy-image-report.html tdk785/boardgame:latest"
            }
        }
        
        stage('Push to Dockerhub') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker push tdk785/boardgame:latest'
                    }   
                }
            }
        }
        
        stage('Deploy to K8S') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'K8S', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.33.138:6443') {
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }
        
        stage('Check the Deployment') {
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'K8S', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://172.31.33.138:6443') {
                    sh "kubectl get pods -n webapps"
                    sh "kubectl get svc -n webapps"
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

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: 'tulshiramcloud@gmail.com',
                from: 'jenkins@example.com',
                replyTo: 'jenkins@example.com',
                mimeType: 'text/html',
                attachmentsPattern: 'trivy-image-report.html'
            )
        }
    }
}
}
