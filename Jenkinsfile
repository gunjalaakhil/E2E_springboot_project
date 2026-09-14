pipeline {
    agent any
    options {
        disableConcurrentBuilds()
    }

    environment {
        APP_NAME   = "E2E_springboot_project"
        DOCKER_REPO = "akhil182/e2e_springboot_project"
        GITOPS_REPO = "https://github.com/gunjalaakhil/E2E_springboot_project.git"
        IMAGE_TAG   = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh '''
                git branch
                git log --oneline -5
                '''
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate(abortPipeline: true)
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                trivy fs . \
                --severity HIGH,CRITICAL \
                --exit-code 0
                '''
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

        stage('Generate Tag') {
            steps {
                script {
                    def imageTag = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()
                    env.IMAGE_TAG = IMAGE_TAG
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${DOCKER_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Image Scan') {
            steps {
                sh "trivy image ${DOCKER_REPO}:${IMAGE_TAG} --severity HIGH,CRITICAL --exit-code 0"
            }
        }

        stage('Push Image') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-creds',
                                                  usernameVariable: 'DOCKER_USER',
                                                  passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${DOCKER_REPO}:${IMAGE_TAG}
                    """
                }
            }
        }
    }
}
