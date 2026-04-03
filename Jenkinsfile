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
