def cleanDockerResources(cleanupTypes) {
    // Loop through the cleanup types selected by the user
    cleanupTypes.each { cleanupType ->
        def pruneCommand = ""

        // Determine the prune command based on selected cleanup type
        if (cleanupType == 'container') {
            pruneCommand = 'docker container prune -f'
        } else if (cleanupType == 'image') {
            pruneCommand = 'docker image prune -a -f'
        } else if (cleanupType == 'volume') {
            pruneCommand = 'docker volume prune -f'
        } else if (cleanupType == 'all') {
            pruneCommand = 'docker system prune -f --volumes'
        }

        // Execute the prune command on the remote Docker host
        withCredentials([sshUserPrivateKey(credentialsId: 'docker_host', 
                                          usernameVariable: 'SSH_USERNAME', 
                                          keyFileVariable: 'SSH_KEY')]) {
            sh """
                ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no $SSH_USERNAME@$REMOTE_HOST \\
                '$pruneCommand'
            """
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
    }

    parameters {
        // String input to let the user input multiple options separated by commas
        string(name: 'CLEANUP_TYPES', defaultValue: 'container,image', description: 'Enter Docker resources to clean (e.g., container,image,volume)')
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
                        -Dsonar.projectName=shopping-cart \
                        -Dsonar.projectKey=shopping-cart \
                        -Dsonar.sources=. \
                        -Dsonar.java.binaries=. \
                        -Dsonar.host.url=http://192.168.1.154:9000/ \
                        -Dsonar.login=28bcc6d0a8390cce56c74fca8697c33b3ee5c4cf

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
                    withDockerRegistry(credentialsId: 'dockerhub_cred', toolName: 'Docker') {
                        sh 'docker build -t shopping-cart -f docker/Dockerfile .'
                        sh 'docker tag shopping-cart scor8709/shopping-cart:latest'
                        sh 'docker push scor8709/shopping-cart:latest'
                    }
                }
            }
        }

        stage('Scanning Docker Image') {
            steps {
                script {
                    sh '''docker run --rm -v $(pwd):/project aquasec/trivy image --format table -o /project/image-scan-report.html scor8709/shopping-cart:latest'''
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
                    withCredentials([sshUserPrivateKey(credentialsId: 'docker_host', 
                                                      usernameVariable: 'SSH_USERNAME', 
                                                      keyFileVariable: 'SSH_KEY')]) {
                        sh """
                            ssh -i "$SSH_KEY" -o StrictHostKeyChecking=no $SSH_USERNAME@$REMOTE_HOST \\
                            'docker run -itd --name ekart -p 8070:8070 scor8709/shopping-cart:latest'
                        """
                    }
                }
            }
        }
    }
}
