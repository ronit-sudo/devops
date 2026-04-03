pipeline {
    agent any

    environment {
        IMAGE = "dockerrr049/nagios-monitor:1.0"
        CONTAINER = "nagios-monitor"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/ronit-sudo/devops.git'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE .'
            }
        }

        stage('Push Image') {
            steps {
                sh 'docker push $IMAGE'
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

            # Encode credentials for DockerHub API
            API_AUTH=$(echo -n "${DOCKERHUB_USR}:${DOCKERHUB_PSW}" | base64)

            # Get all build tags, sort by version, keep older ones
            OLD_TAGS=$(curl -s "https://hub.docker.com/v2/repositories/${IMAGE%:*}/tags/?page_size=100" |
                jq -r '.results[].name' |
                grep build- |
                sort -Vr |
                tail -n +4)

            echo "Tags to delete:"
            echo "$OLD_TAGS"

            for tag in $OLD_TAGS; do
                echo "Deleting tag: $tag"
                curl -s -X DELETE \
                    -H "Authorization: Basic ${API_AUTH}" \
                    "https://hub.docker.com/v2/repositories/${IMAGE%:*}/tags/${tag}/"
            done
            '''
        }
    }
}

        stage('Deploy Container') {
            steps {
                sh '''
                  docker rm -f $CONTAINER || true
                  docker run -d --name $CONTAINER -p 5000:5000 $IMAGE
                '''
            }
        }
    }
}
``
