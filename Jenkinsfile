pipeline {
    agent any

    tools {
        jdk 'jdk17'               // Must match Global Tool name in Jenkins
        nodejs 'node16'           // Must match NodeJS installation name
    }

    environment {
        SCANNER_HOME = tool 'sonarqube-scanner'   // Matches SonarQube Scanner tool name
        DOCKER_IMAGE = 'vamsi213/swiggy-clone'
        DOCKER_TAG = 'latest'
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from GitHub') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Dasarivamsi1/Swiggy-Clone-App1.git',
                    credentialsId: 'github-token'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        //  SonarQube Code Analysis
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh '''
                        echo "🔍 Running SonarQube Analysis..."
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=SwiggyClone \
                        -Dsonar.projectName=SwiggyClone \
                        -Dsonar.projectVersion=1.0 \
                        -Dsonar.sources=. \
                        -Dsonar.sourceEncoding=UTF-8 \
                        -Dsonar.host.url=$SONAR_HOST_URL \
                        -Dsonar.login=$SONAR_AUTH_TOKEN
                    '''
                }
            }
        }

        // SonarQube Quality Gate
        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 4, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        //  Trivy Filesystem Scan
        stage('Trivy FS Scan') {
            steps {
                sh '''
                    echo "🔎 Running Trivy File System Scan..."
                    trivy fs . > trivy-fs-report.txt || true
                '''
                archiveArtifacts artifacts: 'trivy-fs-report.txt', fingerprint: true
            }
        }

        //  Docker Build & Push to DockerHub
        stage('Docker Build & Push') {
            steps {
                script {
                    echo " Building and pushing Docker image..."
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub') {
                        def appImage = docker.build("${DOCKER_IMAGE}:${DOCKER_TAG}")
                        appImage.push()
                    }
                }
            }
        }

        //  Trivy Image Scan
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    echo "🔎 Running Trivy Image Scan..."
                    trivy image ${DOCKER_IMAGE}:${DOCKER_TAG} > trivy-image-report.txt || true
                '''
                archiveArtifacts artifacts: 'trivy-image-report.txt', fingerprint: true
            }
        }

        //  Kubernetes Deployment
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    dir('Kubernetes') {
                        withKubeConfig([credentialsId: 'kubernetes']) {
                            echo "Deploying to Kubernetes Cluster..."
                            sh '''
                                kubectl apply -f deployment.yml
                                kubectl apply -f service.yml
                                kubectl rollout status deployment/swiggy-app
                                kubectl get pods,svc
                            '''
                        }
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker system prune -f || true'
        }
        success {
            echo ' CI/CD Pipeline completed successfully and image pushed to DockerHub!'
        }
        failure {
            echo ' Pipeline failed! Please check Jenkins console for logs.'
        }
    }
}
