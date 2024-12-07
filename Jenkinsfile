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
                    ssh -i "\$SSH_KEY" -o StrictHostKeyChecking=no \$SSH_USERNAME@${params.DOCKER_HOST} \\
                    '${pruneCommand}'
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
        SONAR_URL = params.SONAR_URL // SonarQube server
        SONAR_TOKEN = params.SONAR_TOKEN // Replace with a secure credential
    }

    parameters {
        string(name: 'CLEANUP_TYPES', defaultValue: 'container,image', 
               description: 'Enter Docker resources to clean (e.g., container,image,volume)')
        string(name: 'SONAR_URL', defaultValue: 'http://192.168.1.154:9000/', 
               description: 'Enter the SonarQube server URL')
        string(name: 'SONAR_TOKEN', defaultValue: '', 
               description: 'Enter the SonarQube authentication token')
        string(name: 'SONAR_PROJECT_NAME', defaultValue: 'shopping-cart', 
               description: 'Enter the SonarQube project name')
        string(name: 'SONAR_PROJECT_KEY', defaultValue: 'shopping-cart', 
               description: 'Enter the SonarQube project key')
        string(name: 'DOCKER_HOST', defaultValue: '192.168.1.13', 
               description: 'Enter the Docker Host URL')
        string(name: 'DOCKER_IMAGE_NAME', defaultValue: 'shopping-cart', 
               description: 'Enter the Docker image name')
    }

    stages {
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/DevOpsGuru09/Ekart.git'
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \\
                        -Dsonar.projectName=${params.SONAR_PROJECT_NAME} \\
                        -Dsonar.projectKey=${params.SONAR_PROJECT_KEY} \\
                        -Dsonar.sources=. \\
                        -Dsonar.java.binaries=. \\
                        -Dsonar.host.url=${SONAR_URL} \\
                        -Dsonar.login=${SONAR_TOKEN}
                    """
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
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Scanning Vulnerability') {
            steps {
                sh """
                    docker run --rm -v $(pwd):/project aquasec/trivy fs \\
                    --format table -o /project/fs-report.html /project
                """
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    def dockerImageTag = "${params.DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                    withDockerRegistry(credentialsId: 'dockerhub_cred', toolName: 'Docker') {
                        sh """
                            docker build -t ${params.DOCKER_IMAGE_NAME} -f docker/Dockerfile .
                            docker tag ${params.DOCKER_IMAGE_NAME} scor8709/${dockerImageTag}
                            docker push scor8709/${dockerImageTag}
                        """
                    }
                }
            }
        }

        stage('Scanning Docker Image') {
            steps {
                script {
                    def dockerImageTag = "${params.DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                    sh """
                        docker run --rm -v $(pwd):/project aquasec/trivy image \\
                        --format table -o /project/image-scan-report.html scor8709/${dockerImageTag}
                    """
                }
            }
        }

        stage('Clean Docker Resources') {
            steps {
                script {
                    def cleanupTypes = params.CLEANUP_TYPES.split(',').collect { it.trim() }
                    cleanDockerResources(cleanupTypes)
                }
            }
        }

        stage('Deploy to Docker Container') {
            steps {
                script {
                    def dockerImageTag = "${params.DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                    withCredentials([sshUserPrivateKey(credentialsId: 'docker_host', 
                                                      usernameVariable: 'SSH_USERNAME', 
                                                      keyFileVariable: 'SSH_KEY')]) {
                        sh """
                            ssh -i "\$SSH_KEY" -o StrictHostKeyChecking=no \$SSH_USERNAME@${params.DOCKER_HOST} \\
                            'docker run -itd --name ekart -p 8070:8070 scor8709/${dockerImageTag}'
                        """
                    }
                }
            }
        }
    }
}
