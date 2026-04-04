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
                    echo "Identifying old tags..."

                    API_AUTH=$(echo -n "${DOCKERHUB_USR}:${DOCKERHUB_PSW}" | base64)

                    ALL_TAGS=$(curl -s "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/?page_size=100" |
                        jq -r '.results[].name' |
                        grep build- |
                        sort -Vr)

                    OLD_TAGS=$(echo "$ALL_TAGS" | tail -n +4)

                    echo "Tags to delete:"
                    echo "$OLD_TAGS"

                    for tag in $OLD_TAGS; do
                        echo "Deleting tag: $tag"
                        curl -s -X DELETE \
                          -H "Authorization: Basic ${API_AUTH}" \
                          "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/${tag}/"
                    done
                    '''
                }
            }
        }

        stage('Deploy Container') {
            steps {
                sh '''
                echo "Deploying container..."

                docker rm -f ${CONTAINER} || true
                docker run -d --name ${CONTAINER} -p 5000:5000 ${IMAGE_NAME}:build-${BUILD_NUMBER}
                '''
            }
        }
    }
}
