pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    // Fires when the GitHub webhook (Step 7) hits http://<jenkins-ip>:8080/github-webhook/
    triggers {
        githubPush()
    }

    environment {
        // ---- EDIT THESE ----
        AWS_ACCOUNT_ID = '414964671989'
        AWS_REGION     = 'us-east-1'
        // Vite bakes this into the frontend bundle at build time
        VITE_API_URL   = 'http://localhost:4000'
        // --------------------

        BACKEND_IMAGE  = 'food-delivery-backend'
        FRONTEND_IMAGE = 'food-delivery-frontend'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    }

    stages {

        stage('Checkout') {
            steps {
                // Pulls the repo and branch configured on the Jenkins job
                checkout scm
                sh 'git rev-parse --abbrev-ref HEAD'
                sh 'git log -1 --oneline'
            }
        }

        // The app builds (npm install / vite build) happen INSIDE the multi-stage
        // Dockerfiles, so Jenkins itself does not need Node installed.
        stage('Build Backend Image') {
            steps {
                sh 'docker build -t $BACKEND_IMAGE:$IMAGE_TAG ./backend'
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh 'docker build --build-arg VITE_API_URL=$VITE_API_URL -t $FRONTEND_IMAGE:$IMAGE_TAG ./frontend'
            }
        }

        stage('Sanity Check') {
            steps {
                // Confirms the hardening from Task 5: containers must not run as root
                sh 'docker run --rm --entrypoint whoami $BACKEND_IMAGE:$IMAGE_TAG'
                sh 'docker run --rm --entrypoint whoami $FRONTEND_IMAGE:$IMAGE_TAG'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DH_USER',
                    passwordVariable: 'DH_PASS'
                )]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        for IMG in $BACKEND_IMAGE $FRONTEND_IMAGE; do
                            docker tag $IMG:$IMAGE_TAG $DH_USER/$IMG:$IMAGE_TAG
                            docker tag $IMG:$IMAGE_TAG $DH_USER/$IMG:latest
                            docker push $DH_USER/$IMG:$IMAGE_TAG
                            docker push $DH_USER/$IMG:latest
                            docker manifest inspect $DH_USER/$IMG:$IMAGE_TAG > /dev/null
                            echo "Verified on Docker Hub: $DH_USER/$IMG:$IMAGE_TAG"
                        done
                        docker logout
                    '''
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                // If the EC2 instance has an IAM role with ECR permissions,
                // delete this withCredentials wrapper and keep only the sh block.
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds',
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                        for IMG in $BACKEND_IMAGE $FRONTEND_IMAGE; do
                            aws ecr describe-repositories --repository-names $IMG --region $AWS_REGION > /dev/null 2>&1 \
                                || aws ecr create-repository --repository-name $IMG --region $AWS_REGION
                            docker tag $IMG:$IMAGE_TAG $ECR_REGISTRY/$IMG:$IMAGE_TAG
                            docker tag $IMG:$IMAGE_TAG $ECR_REGISTRY/$IMG:latest
                            docker push $ECR_REGISTRY/$IMG:$IMAGE_TAG
                            docker push $ECR_REGISTRY/$IMG:latest
                            aws ecr describe-images --repository-name $IMG --image-ids imageTag=$IMAGE_TAG --region $AWS_REGION > /dev/null
                            echo "Verified on ECR: $ECR_REGISTRY/$IMG:$IMAGE_TAG"
                        done
                        docker logout $ECR_REGISTRY
                    '''
                }
            }
        }
    }

    post {
        always {
            // Keep the EC2 disk from filling up with old layers
            sh 'docker image prune -f || true'
        }
        success {
            echo "Pipeline succeeded: images tagged ${IMAGE_TAG} and latest are on Docker Hub and ECR."
        }
        failure {
            echo 'Pipeline failed. Check the stage that turned red in the console output.'
        }
    }
}
