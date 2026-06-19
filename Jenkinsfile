pipeline {
    agent any

    environment {
        // Use your DockerHub username
        IMAGE_NAME = "ezzine-mouhamed/student-management"
        IMAGE_TAG = "1.0.${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = "student-management"
        KUBE_NAMESPACE = "devops"
    }

    stages {

        stage('Clone Repository') {
            steps {
                echo "📥 Cloning repository..."
                git branch: 'main',
                    url: 'https://github.com/ezzine-mouhamed/student-management.git'
                sh 'ls -la'
            }
        }

        stage('Compile & Test') {
            steps {
                echo "⚙️ Compiling with Maven..."
                sh 'mvn clean compile'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_AUTH_TOKEN')]) {
                        sh """
                            mvn sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=$SONAR_AUTH_TOKEN
                        """
                    }
                }
            }
        }

        stage('Package Application') {
            steps {
                echo "📦 Packaging (skipping tests)..."
                sh 'mvn package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "🐳 Building Docker image..."
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Push Docker Image') {
            steps {
                echo "⬆️ Pushing to DockerHub..."
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        docker push $IMAGE_NAME:$IMAGE_TAG
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo "🚀 Updating Kubernetes deployment..."
                sh """
                    kubectl set image deployment/student-management \
                    student-management=${IMAGE_NAME}:${IMAGE_TAG} \
                    -n ${KUBE_NAMESPACE}
                """
                sh """
                    kubectl rollout status deployment/student-management \
                    -n ${KUBE_NAMESPACE} --timeout=120s
                """
                sh "kubectl get pods -n ${KUBE_NAMESPACE}"
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline Succeeded!"
            script {
                def node_ip = sh(script: "kubectl get node -o jsonpath='{.items[0].status.addresses[?(@.type==\"InternalIP\")].address}'", returnStdout: true).trim()
                echo "🌐 Application URL: http://${node_ip}:30089/student/"
                echo "🌐 SonarQube: http://${node_ip}:32000"
            }
        }
        failure {
            echo "❌ Pipeline Failed – check console logs"
        }
    }
}
