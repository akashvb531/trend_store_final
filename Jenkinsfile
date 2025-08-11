pipeline {
  agent any

  environment {
    // change this to your real DockerHub username/repo
    DOCKER_IMAGE = 'akashvb/trend-static-app:latest'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${DOCKER_IMAGE} ."
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'docker-hub',         // must exist in Jenkins
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_IMAGE}
            docker logout
          '''
        }
      }
    }
  }

  post {
    always {
      // optional clean-up
      sh 'docker image ls | head -n 10 || true'
    }
  }
}

