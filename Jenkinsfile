pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'satyasaia99'
        DOCKERHUB_IMAGE = 'hotstar'
        DOCKERHUB_TAG = 'v2'
        SONARQUBE_ENV = 'sq'
        NEXUS_REPO = 'maven-releases'
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/SatyasaiA99/hotstarjava.git'
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Cleanup Old Docker Artifacts') {
            steps {
                sh '''
                echo "🧹 Cleaning up old containers and images..."
                docker rm -f $(docker ps -aq --filter "name=myapp-cont") || true
                docker rmi -f ${DOCKERHUB_USER}/${DOCKERHUB_IMAGE}:${DOCKERHUB_TAG} || true
                '''
            }
        }

        stage('JENKINS TO NEXUS') {
            steps {
                withMaven(globalMavenSettingsConfig: '64516d7d-cd77-4963-a65e-da48f786f6ef', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                    sh 'mvn deploy'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKERHUB_USER}/${DOCKERHUB_IMAGE}:${DOCKERHUB_TAG} ."
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'Docker',
                                                 usernameVariable: 'DOCKER_USER',
                                                 passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${DOCKERHUB_USER}/${DOCKERHUB_IMAGE}:${DOCKERHUB_TAG}
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                sh '''
                echo "🚀 Deploying to Kubernetes..."

                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml

                kubectl rollout status deployment/hotstar-deployment
                '''
            }
        }
    }
}

    post {
        always {
            sh '''
            echo "🧹 Final cleanup..."
            docker system prune -af || true
            '''
        }
    }
}
