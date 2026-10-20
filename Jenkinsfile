pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'YOUR_DOCKERHUB_USERNAME'   // <-- change this
        IMAGE_NAME     = "${DOCKERHUB_USER}/airline-booking-demo"
        IMAGE_TAG      = "${BUILD_NUMBER}"
    }

    triggers {
        pollSCM('H/2 * * * *')   // checks GitHub every ~2 min and runs on new commits
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build and Test') {
            // Static site: "build" = validate the files, "test" = syntax/structure checks
            steps {
                sh '''
                    set -e
                    test -s index.html
                    grep -q "<!DOCTYPE html>" index.html
                    grep -q "<title>" index.html
                    grep -q "</html>" index.html
                    echo "index.html checks passed"
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t $IMAGE_NAME:$IMAGE_TAG -t $IMAGE_NAME:latest .
                    docker run --rm $IMAGE_NAME:$IMAGE_TAG nginx -t
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push $IMAGE_NAME:$IMAGE_TAG
                        docker push $IMAGE_NAME:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh '''
                    sed -i "s|image: .*|image: $IMAGE_NAME:$IMAGE_TAG|" k8s/deployment.yaml
                    kubectl apply -f k8s/
                    kubectl rollout status deployment/airline-booking --timeout=120s
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    kubectl get pods -l app=airline-booking -o wide
                    kubectl get deployment airline-booking
                    kubectl get svc airline-booking-service
                    URL=$(minikube service airline-booking-service --url)
                    echo "App URL inside the server: $URL"
                    curl -sf "$URL" | grep -i "Skyline Air"
                '''
            }
        }
    }

    post {
        success { echo "Deployed ${IMAGE_NAME}:${IMAGE_TAG} to Minikube" }
        failure { echo 'Pipeline failed - read the log of the red stage' }
    }
}
