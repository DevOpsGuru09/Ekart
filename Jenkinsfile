// Function to clean Docker resources
def cleanDockerResources(cleanupTypes) {
    // Map cleanup types to their respective Docker prune commands
    def pruneCommands = [
        container: 'docker container prune -f',
        image    : 'docker image prune -a -f',
        volume   : 'docker volume prune -f',
        all      : 'docker system prune -f --volumes'
    ]

    cleanupTypes.each { cleanupType ->
        def pruneCommand = pruneCommands[cleanupType]
        if (pruneCommand) {
            // Execute the prune command on the remote Docker host
            withCredentials([sshUserPrivateKey(credentialsId: 'docker_host', 
                                              usernameVariable: 'SSH_USERNAME', 
                                              keyFileVariable: 'SSH_KEY')]) {
                sh """
                    ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no $SSH_USERNAME@$REMOTE_HOST \ 
                    '$pruneCommand'
                """
            }
        } else {
            echo "Invalid cleanup type: ${cleanupType}. Skipping..."
        }
    }
}


pipeline {
    agent any
    tools {
        maven 'MAVEN'
    }

    environment {
        SCANNER_HOME = tool 'SONARQUBE'
        REMOTE_HOST = '192.168.1.13' // Replace with your Docker host IP
        // // REMOTE_HOST = params.DOCKER_HOST // Docker host IP
        // SONAR_URL = params.SONAR_URL // SonarQube server
        // SONAR_TOKEN = params.SONAR_TOKEN
    }

    parameters {
        // String input to let the user input multiple options separated by commas
        string(name: 'CLEANUP_TYPES', defaultValue: 'container,image', description: 'Enter Docker resources to clean (e.g., container,image,volume)')
        // Input for SonarQube details
        string(name: 'SONAR_URL', defaultValue: 'http://192.168.1.154:9000/', description: 'Enter the SonarQube server URL')
        string(name: 'SONAR_TOKEN', defaultValue: '28bcc6d0a8390cce56c74fca8697c33b3ee5c4cf', description: 'Enter the SonarQube authentication token')
        string(name: 'SONAR_PROJECT_NAME', defaultValue: 'shopping-cart', description: 'Enter the SonarQube project name')
        string(name: 'SONAR_PROJECT_KEY', defaultValue: 'shopping-cart', description: 'Enter the SonarQube project key')
        // Input for Docker details
        // string(name: 'DOCKER_HOST', defaultValue: '192.168.1.13', description: 'Enter the Docker Host URL')
        string(name: 'DOCKER_IMAGE_NAME', defaultValue: 'shopping-cart', description: 'Enter the Docker image name')
    }


    stages {   
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/DevOpsGuru09/Ekart.git'
            }
        }     
        stage('Compile') {
            steps {
                script {
                    sh 'mvn clean compile'
                }
            }
        }
        stage('SonarQube Analysis') {
            steps {
                script {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=${SONAR_PROJECT_NAME} \
                        -Dsonar.projectKey=${SONAR_PROJECT_NAME} \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=. \
                        -Dsonar.host.url=${SONAR_URL} \
                        -Dsonar.login=${SONAR_TOKEN}

                    '''
                }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                    dependencyCheck additionalArguments: '-s ./', odcInstallation: 'DP-CHECK'
                      dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Build Application') {
            steps {
                script {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Scanning Vulnerability') {
            steps {
                script {
                    sh 'docker run --rm -v $(pwd):/project aquasec/trivy fs --format table -o /project/fs-report.html /project'
                }
            }
        }
        
        stage('Build & Push to Docker') {
            steps {
                script {
                    def dockerTag = "${env.DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                    withDockerRegistry(credentialsId: 'dockerhub_cred', toolName: 'Docker') {
                        sh 'docker build -t ${dockerTag} -f docker/Dockerfile .'
                        sh 'docker tag ${dockerTag} scor8709/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}'
                        sh 'docker push scor8709/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}'
                    }
                }
            }
        }

        stage('Scanning Docker Image') {
            steps {
                script {
                    sh '''docker run --rm -v $(pwd):/project aquasec/trivy image --format table -o /project/image-scan-report.html scor8709/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}'''
                }
            }
        }

        stage('Clean Docker Resources') {
            steps {
                script {
                    // Split the input into a list and call the cleanup function
                    def cleanupTypes = params.CLEANUP_TYPES.split(',')
                    cleanDockerResources(cleanupTypes)
                }
            }
        }
        
        stage('Deploy to Docker Container') {
            steps {
                script {
                    def dockerTag = "${env.DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                    withCredentials([sshUserPrivateKey(credentialsId: 'docker_host', 
                                                      usernameVariable: 'SSH_USERNAME', 
                                                      keyFileVariable: 'SSH_KEY')]) {
                        sh """
                            ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no $SSH_USERNAME@$REMOTE_HOST \\
                            'docker run -itd --name ekart -p 8070:8070 scor8709/${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}'
                        """
                    }
                }
            }
        }
    }
}
