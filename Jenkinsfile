pipeline {
    agent any
    tools {
        maven 'MAVEN'
    }

    environment {
        SCANNER_HOME = tool 'SONAR_SCAN'
        REMOTE_HOST = '192.168.1.9' // Replace with your Docker host IP
    }
    stages {   
        stage('Checkout SCM') {
            steps {
                git branch: 'main', url: 'https://github.com/jaiswaladi246/Ekart'
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
                        -Dsonar.host.url=http://192.168.1.15:9002/ \
                        -Dsonar.login=squ_9b69eab749e454e3b6b15de42501a67bd99412ca

                    '''
                }
            }
        }
        stage('OWASP Dependency Check') {
            steps {
                    dependencyCheck additionalArguments: '-s ./', odcInstallation: 'DP-Check'
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
                    withDockerRegistry(credentialsId: '48355594-1782-4bc9-b2d7-3470ed1322bb', toolName: 'Docker') {
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
                    withCredentials([usernamePassword(credentialsId: '0ab2a435-eead-4006-a185-e907755fd565', 
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
