pipeline {
    agent any
    tools {
        maven 'MAVEN'
    }

    environment {
        SCANNER_HOME = tool 'SONARQUBE'
        REMOTE_HOST = '192.168.1.13' // Replace with your Docker host IP
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
        stage('Deploy Container') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker_id', 
                                                      usernameVariable: 'SSH_USERNAME', 
                                                      passwordVariable: 'SSH_PASSWORD')]) {
                        sh """
                            sshpass -p "$SSH_PASSWORD" ssh -o StrictHostKeyChecking=no $SSH_USERNAME@$REMOTE_HOST \\
                            'docker run -itd --name ekart -p 8070:8070 scor8709/shopping-cart:latest'
                        """
                    }
                }
            }
        }
    }
}
