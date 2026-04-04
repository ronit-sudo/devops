pipeline {
    agent any

    environment {
        DOCKER_USER    = "dockerrr049"
        IMAGE_REPO     = "myapp"
        CONTAINER_NAME = "myapp"
        KEEP_IMAGES    = 3
    }

    stages {

        stage('Build Docker Image') {
            steps {
                script {
                    // Zero-pad Jenkins build number (example: 19 → 019)
                    PADDED = String.format("%03d", BUILD_NUMBER.toInteger())
                    env.IMAGE_TAG = "build-${PADDED}"
                }
                sh '''
echo "Building image ${DOCKER_USER}/${IMAGE_REPO}:${IMAGE_TAG}"
docker build -t ${DOCKER_USER}/${IMAGE_REPO}:${IMAGE_TAG} .
'''
            }
        }

        stage('Docker Login (Push Auth)') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_LOGIN_USER',
                        passwordVariable: 'DOCKER_LOGIN_PASS'
                    )
                ]) {
                    sh '''
echo "$DOCKER_LOGIN_PASS" | docker login -u "$DOCKER_LOGIN_USER" --password-stdin
'''
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                sh '''
docker push ${DOCKER_USER}/${IMAGE_REPO}:${IMAGE_TAG}
'''
            }
        }

        stage('Deploy Container') {
            steps {
                sh '''
docker stop ${CONTAINER_NAME} || true
docker rm ${CONTAINER_NAME} || true

docker run -d \
  --name ${CONTAINER_NAME} \
  -p 5050:2020 \
  ${DOCKER_USER}/${IMAGE_REPO}:${IMAGE_TAG}

echo "Deployed ${DOCKER_USER}/${IMAGE_REPO}:${IMAGE_TAG}"
'''
            }
        }

        stage('Cleanup Old DockerHub Tags (Keep Only 3)') {
            steps {
                withCredentials([
                    string(credentialsId: 'dockerhub-token', variable: 'DOCKER_PAT')
                ]) {
                    sh '''
echo "Generating DockerHub JWT token..."

JWT_RESPONSE=$(curl -s -X POST https://hub.docker.com/v2/users/login/ \
  -H "Content-Type: application/json" \
  -d "{
        \\"username\\": \\"${DOCKER_USER}\\",
        \\"password\\": \\"${DOCKER_PAT}\\"
      }")

JWT_TOKEN=$(echo "$JWT_RESPONSE" | jq -r .token)

echo "JWT Token Received"

echo "Fetching all tags"
TAGS=$(curl -s -H "Authorization: Bearer $JWT_TOKEN" \
https://hub.docker.com/v2/repositories/${DOCKER_USER}/${IMAGE_REPO}/tags/?page_size=100 \
| jq -r '.results | sort_by(.last_updated) | reverse | .[].name')

COUNT=0
for TAG in $TAGS; do
    COUNT=$((COUNT+1))
    if [ $COUNT -gt $KEEP_IMAGES ]; then
        echo "Deleting old tag: $TAG"
        curl -s -X DELETE \
            -H "Authorization: Bearer $JWT_TOKEN" \
            https://hub.docker.com/v2/repositories/${DOCKER_USER}/${IMAGE_REPO}/tags/$TAG/
    else
        echo "Keeping tag: $TAG"
    fi
done
'''
                }
            }
        }

    }

    post {
        success {
            echo "✔ Pipeline Success — Built, deployed, and cleaned up old images"
        }
        failure {
            echo "❌ Pipeline Failed — Check logs"
        }
    }
}
