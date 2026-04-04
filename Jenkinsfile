pipeline {
    agent any

    environment {
        IMAGE_NAME = "dockerrr049/nagios-monitor"
        CONTAINER  = "nagios-monitor"
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
                echo "Building Image: ${IMAGE_NAME}:build-${BUILD_NUMBER}"
                docker build -t ${IMAGE_NAME}:build-${BUILD_NUMBER} .
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USR',
                    passwordVariable: 'DOCKERHUB_PSW'
                )]) {
                    sh '''
                    echo "$DOCKERHUB_PSW" | docker login -u "$DOCKERHUB_USR" --password-stdin
                    docker push ${IMAGE_NAME}:build-${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Cleanup Old Tags (Keep last 3)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USR',
                    passwordVariable: 'DOCKERHUB_PSW'
                )]) {

                    sh '''
                    echo "Cleaning older DockerHub tags and manifests..."

                    API_AUTH=$(echo -n "${DOCKERHUB_USR}:${DOCKERHUB_PSW}" | base64)

                    ALL_MANIFESTS=$(curl -s "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/?page_size=100" |
                        jq -r '.results[] | select(.name | startswith("build-")) | (.name + " " + .digest)' |
                        sort -Vr)

                    echo "Found build tags:"
                    echo "$ALL_MANIFESTS"

                    OLD_MANIFESTS=$(echo "$ALL_MANIFESTS" | tail -n +4 | awk '{print $2}')

                    echo "Deleting these manifests:"
                    echo "$OLD_MANIFESTS"

                    for digest in $OLD_MANIFESTS; do
                        echo "Deleting manifest: $digest"
                        curl -s -X DELETE \
                          -H "Authorization: Basic ${API_AUTH}" \
                          "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/manifests/${digest}/"
                    done
                    '''
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh '''
                echo "Deploying Container..."

                docker rm -f ${CONTAINER} || true
                docker run -d --name ${CONTAINER} -p 5000:5000 ${IMAGE_NAME}:build-${BUILD_NUMBER}

                echo "Container deployed successfully."
                '''
            }
        }
    }
}
