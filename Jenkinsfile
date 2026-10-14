pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    // Fires when the GitHub webhook hits http://3.81.165.236:8080/github-webhook/
    triggers {
        githubPush()
    }

    environment {
        // ---- EDIT THIS ----
        // Vite bakes this into the frontend bundle at build time.
        // Use the public address the BROWSER will use to reach the backend.
        VITE_API_URL   = 'http://3.81.165.236:4000'
        // -------------------

        BACKEND_IMAGE  = 'food-delivery-backend'
        FRONTEND_IMAGE = 'food-delivery-frontend'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
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

        stage('Push to GHCR') {
            steps {
                // Reuses the github-creds credential. Its token needs the
                // write:packages and read:packages scopes (not just repo).
                withCredentials([usernamePassword(
                    credentialsId: 'github-creds',
                    usernameVariable: 'GH_USER',
                    passwordVariable: 'GH_TOKEN'
                )]) {
                    sh '''
                        # GHCR image paths must be lowercase
                        GH_OWNER=$(echo "$GH_USER" | tr '[:upper:]' '[:lower:]')
                        echo "$GH_TOKEN" | docker login ghcr.io -u "$GH_USER" --password-stdin
                        for IMG in $BACKEND_IMAGE $FRONTEND_IMAGE; do
                            docker tag $IMG:$IMAGE_TAG ghcr.io/$GH_OWNER/$IMG:$IMAGE_TAG
                            docker tag $IMG:$IMAGE_TAG ghcr.io/$GH_OWNER/$IMG:latest
                            docker push ghcr.io/$GH_OWNER/$IMG:$IMAGE_TAG
                            docker push ghcr.io/$GH_OWNER/$IMG:latest
                            docker manifest inspect ghcr.io/$GH_OWNER/$IMG:$IMAGE_TAG > /dev/null
                            echo "Verified on GHCR: ghcr.io/$GH_OWNER/$IMG:$IMAGE_TAG"
                        done
                        docker logout ghcr.io
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
            echo "Pipeline succeeded: images tagged ${IMAGE_TAG} and latest are on Docker Hub and GHCR."
        }
        failure {
            echo 'Pipeline failed. Check the stage that turned red in the console output.'
        }
    }
}