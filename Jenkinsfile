pipeline {
    agent any

    environment {
        DOCKER_USER = 'your-dockerhub-username'
        DOCKER_PASS = credentials('dockerhub-token') // Jenkins secret
    }

    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_USER/trend-static-app .'
            }
        }

        stage('Push to DockerHub') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub-token', variable: 'DOCKER_PASS')]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    sh 'docker push $DOCKER_USER/trend-static-app'
                }
            }
        }
    }
}
