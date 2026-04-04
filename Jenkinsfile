pipeline {
    agent any

    environment {
        DOCKER_USER    = "dockerrr049"
        IMAGE_REPO     = "myapp"
        CONTAINER_NAME = "myapp"
        KEEP_IMAGES    = 3
    }

    stages {

        stage('Show Build Info') {
            steps {
                echo "🚀 Jenkins Build Number = ${BUILD_NUMBER}"
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
echo "📦 Building image ${DOCKER_USER}/${IMAGE_REPO}:${BUILD_NUMBER}"
docker build -t ${DOCKER_USER}/${IMAGE_REPO}:${BUILD_NUMBER} .
'''
            }
        }

        stage('Docker Login') {
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
echo "⬆️  Pushing image ${DOCKER_USER}/${IMAGE_REPO}:${BUILD_NUMBER}"
docker push ${DOCKER_USER}/${IMAGE_REPO}:${BUILD_NUMBER}
'''
            }
        }

        stage('Deploy Latest Image') {
            steps {
                sh '''
echo "🚀 Deploying image ${DOCKER_USER}/${IMAGE_REPO}:${BUILD_NUMBER}"

docker pull ${DOCKER_USER}/${IMAGE_REPO}:${BUILD_NUMBER}
docker stop ${CONTAINER_NAME} || true
docker rm ${CONTAINER_NAME} || true

docker run -d \
  --name ${CONTAINER_NAME} \
  -p 5050:2020 \
  ${DOCKER_USER}/${IMAGE_REPO}:${BUILD_NUMBER}

echo "✅ Deployment Complete"
'''
            }
        }

        stage('Cleanup Old DockerHub Tags (Keep Last 3 Only)') {
            steps {
                withCredentials([
                    string(credentialsId: 'dockerhub-token', variable: 'DOCKER_PAT')
                ]) {
                    sh '''
echo "🧹 Cleaning Old DockerHub Tags (Keep only ${KEEP_IMAGES})"

echo "Generating DockerHub JWT token..."

JWT_RESPONSE=$(curl -s -X POST https://hub.docker.com/v2/users/login/ \
  -H "Content-Type: application/json" \
  -d "{
        \\"username\\": \\"${DOCKER_USER}\\",
        \\"password\\": \\"${DOCKER_PAT}\\"
      }")

JWT_TOKEN=$(echo "$JWT_RESPONSE" | jq -r .token)

if [ -z "$JWT_TOKEN" ] || [ "$JWT_TOKEN" = "null" ]; then
  echo "❌ ERROR: DockerHub JWT generation failed"
  exit 1
fi

echo "✅ JWT token generated"

# Fetch tags sorted by last updated
TAGS=$(curl -s -H "Authorization: Bearer $JWT_TOKEN" \
  https://hub.docker.com/v2/repositories/${DOCKER_USER}/${IMAGE_REPO}/tags/?page_size=100 \
  | jq -r '.results | sort_by(.last_updated) | reverse | .[].name')

COUNT=0
for TAG in $TAGS; do
  COUNT=$((COUNT+1))
  if [ $COUNT -gt $KEEP_IMAGES ]; then
      echo "🗑 Deleting old tag: $TAG"
      curl -s -X DELETE \
        -H "Authorization: Bearer $JWT_TOKEN" \
        https://hub.docker.com/v2/repositories/${DOCKER_USER}/${IMAGE_REPO}/tags/$TAG/
  else
      echo "✔ Keeping tag: $TAG"
  fi
done
'''
                }
            }
        }
    }

    post {
        success {
            echo "🎉 SUCCESS: Build ${BUILD_NUMBER} deployed & cleanup complete"
        }
        failure {
            echo "❌ FAILURE: Check Jenkins logs"
        }
    }
}
