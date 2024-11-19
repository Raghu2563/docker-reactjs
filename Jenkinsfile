pipeline {
    agent any

    tools {
        nodejs 'Node JS 9.6.1'  // Make sure NodeJS is configured in Jenkins
    }

    environment {
        NEXUS_URL = '13.201.32.66:8081' 
        NEXUS_CREDENTIALS = credentials('nexus-credentials')
        SONARQUBE_SERVER = 'SonarQube'  // Exactly as in your previous project
        DOCKER_IMAGE = 'react-app'
        DOCKER_REPO = "raghu2563/react-app"
        VERSION = "1.0.${BUILD_NUMBER}"
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
    }

    parameters {
        booleanParam(name: 'PROD_BUILD', defaultValue: true, description: 'Enable this as a production build')
    }

    stages {
        stage('Source') {
            steps {
                git branch: 'master', changelog: false, credentialsId: 'gh-up', poll: false, url: 'https://github.com/Raghu2563/docker-reactjs.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarScanner' // Ensure 'SonarScanner' is configured in Global Tool Configuration
                    withSonarQubeEnv('SonarQube') { // Use the configured SonarQube environment
                        withCredentials([string(credentialsId: 'sonar_token1', variable: 'SONAR_TOKEN')]) {
                            sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=react-app -Dsonar.sources=. -Dsonar.host.url=http://13.234.136.24:9000 -Dsonar.login=${SONAR_TOKEN}"
                        }
                    }
                }
            }
        } 

        stage('Test') {
            parallel {
                stage('Unit Test') {
                    steps {
                        echo 'Running Unit tests'
                    }
                }
                stage('Integration Test') {
                    steps {
                        echo 'Running integration tests'
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        docker build -t 13.201.32.66:8085/repository/docker-hosted/react-app:${VERSION} \
                        -t ${DOCKER_REPO}:${VERSION} \
                        -t ${DOCKER_REPO}:latest .
                    """
                }
            }
        }
        
        stage('Publish to Nexus') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'nexus-credentials', 
                        usernameVariable: 'NEXUS_USER', 
                        passwordVariable: 'NEXUS_PASS')]) {
                        sh "docker login -u ${NEXUS_USER} -p ${NEXUS_PASS} 13.201.32.66:8085"
                        sh "docker push 13.201.32.66:8085/repository/docker-hosted/${DOCKER_IMAGE}:${VERSION}"
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    // Login to Docker Hub
                    sh 'echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                    
                    // Push both version-tagged and latest images
                    sh """
                        docker push ${DOCKER_REPO}:${VERSION}
                        docker push ${DOCKER_REPO}:latest
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            when {
                expression { return params.PROD_BUILD }
            }
            steps {
                withCredentials([sshUserPrivateKey(credentialsId: 'pk_react_app', 
                    keyFileVariable: 'SSHKEY', 
                    usernameVariable: 'USER')]) {
                    sh '''
                        # Copy docker-compose file to EC2
                        rsync -avzP -e "ssh -o StrictHostKeyChecking=no -i $SSHKEY" \
                        docker-compose.yml ${USER}@15.207.84.222:/home/ec2-user/react-app

                        # Update and restart containers
                        ssh -o StrictHostKeyChecking=no -i $SSHKEY ${USER}@15.207.84.222 \
                        "cd /home/ec2-user/react-app && \
                        docker-compose down && \
                        docker login -u $DOCKERHUB_CREDENTIALS_USR -p $DOCKERHUB_CREDENTIALS_PSW && \
                        docker-compose pull && \
                        docker-compose up -d"
                    '''
                }
            }
        }
    }

    post {
        always {
            script {
                echo "Cleaning up Docker images..."

                // Try to remove specific images and suppress errors if the image doesn't exist
                try {
                    sh "docker rmi ${DOCKER_REPO}:${VERSION} || true"
                    sh "docker rmi ${DOCKER_REPO}:latest || true"
                    sh "docker rmi ${NEXUS_URL}/${DOCKER_IMAGE}:${VERSION} || true"
                    sh "docker rmi ${DOCKER_IMAGE}:${VERSION} || true"
                } catch (Exception e) {
                    echo "Error during image removal: ${e.getMessage()}"
                }

                // Logout from Nexus and Docker Hub registries
                sh "docker logout ${NEXUS_URL} || true"
                sh "docker logout || true"

                // Clean up the workspace after the build
                cleanWs()
            }
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
