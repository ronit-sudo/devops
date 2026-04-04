pipeline {
    agent any

    environment {
        IMAGE_NAME = "dockerrr049/myapp"
        CONTAINER  = "myapp"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ronit-sudo/devops.git'
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                echo "Building: ${IMAGE_NAME}:build-${BUILD_NUMBER}"
                docker build -t ${IMAGE_NAME}:build-${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                    docker push ${IMAGE_NAME}:build-${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Cleanup Old Tags (Keep last 3)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                    echo "======== CLEANUP STARTED ========"

                    API_AUTH=$(echo -n "${DOCKER_USER}:${DOCKER_PASS}" | base64)

                    echo "Fetching tags…"
                    ALL=$(curl -s "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/?page_size=100" |
                      jq -r '.results[] | select(.name | startswith("build-")) | (.name + " " + .digest)' |
                      sort -Vr)

                    echo "All build tags:"
                    echo "$ALL"

                    OLD=$(echo "$ALL" | tail -n +4 | awk '{print $2}')

                    echo "Deleting manifests:"
                    echo "$OLD"

                    for digest in $OLD; do
                        echo "Deleting manifest: $digest"
                        curl -s -X DELETE \
                          -H "Authorization: Basic ${API_AUTH}" \
                          -H "Accept: application/vnd.docker.distribution.manifest.list.v2+json" \
                          "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/manifests/${digest}/"
                    done

                    echo "======== CLEANUP DONE ========"
                    '''
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh '''
                echo "Deploying latest build…"

                docker rm -f ${CONTAINER} || true
                docker run -d --name ${CONTAINER} -p 5050:2020 ${IMAGE_NAME}:build-${BUILD_NUMBER}

                echo "Deployment complete."
                '''
            }
        }

    }
}
