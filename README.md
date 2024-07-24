pipeline {
    agent any
    tools {
        jdk 'jdk17'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/cyberdesk07/a-swiggy-clone.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix
                    '''
                }
            }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    def imageTag = "cyberdesk07/zamato1:${BUILD_NUMBER}"
                    withDockerRegistry(credentialsId: 'dockerhub', toolName: 'docker') {   
                        sh "docker build -t ${imageTag} ."
                        sh "docker push ${imageTag}"
                    }
                }
            }
        }
        stage('Trivy Scan') {
            steps {
                script {
                    def imageTag = "cyberdesk07/zamato1:${BUILD_NUMBER}"
                    // Install Trivy to the workspace directory
                    sh 'curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b $WORKSPACE'
                    
                    // Add Trivy to PATH and run Trivy scan
                    withEnv(["PATH+TRIVY=$WORKSPACE"]) {
                        sh "trivy image ${imageTag}"
                    }
                }
            }
        }
        stage('Update Deployment File') {
            steps {
                script {
                    withCredentials([string(credentialsId: 'GITHUB_TOKEN', variable: 'GITHUB_TOKEN')]) {
                       def NEW_IMAGE_NAME = "cyberdesk07/zamato1:${BUILD_NUMBER}"   // Update your image here
                       sh "sed -i 's|image: .*|image: ${NEW_IMAGE_NAME}|' Kubernetes/deployment.yml"
                       sh 'git config --global user.email "cyberdesk07@gmail.com"'
                       sh 'git config --global user.name "cyberdesk07"'
                       sh 'git add Kubernetes/deployment.yml'
                       sh "git commit -m 'Update deployment image to ${NEW_IMAGE_NAME}' || echo 'No changes to commit'"
                       sh "git push https://${GITHUB_TOKEN}@github.com/cyberdesk07/a-swiggy-clone.git HEAD:main"
                    }
                }
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
