pipeline {
    agent any

    environment {
        IMAGE_NAME = "dockerrr049/myapp"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def tag = "build-${BUILD_NUMBER}"
                    env.IMAGE_TAG = tag
                }
                sh '''
                echo "Building ${IMAGE_NAME}:${IMAGE_TAG}"
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Login & Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USR',
                    passwordVariable: 'DOCKERHUB_PSW'
                )]) {
                    sh '''
                    echo "$DOCKERHUB_PSW" | docker login -u "$DOCKERHUB_USR" --password-stdin
                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Cleanup Old Tags (keep last 3)') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKERHUB_USR',
                    passwordVariable: 'DOCKERHUB_PSW'
                )]) {
                    sh '''
                    echo "Cleaning old tags..."

                    OLD_TAGS=$(curl -s "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/?page_size=100" |
                        jq -r '.results[].name' |
                        grep build- |
                        sort -Vr |
                        tail -n +4)

                    for tag in $OLD_TAGS; do
                        echo "Deleting tag: $tag"
                        curl -s -X DELETE \
                            -u "${DOCKERHUB_USR}:${DOCKERHUB_PSW}" \
                            "https://hub.docker.com/v2/repositories/${IMAGE_NAME}/tags/$tag/"
                    done
                    '''
                }
            }
        }
    }
}
