pipeline {
  agent any

  parameters {
    string(name: 'AWS_REGION',   defaultValue: 'ap-south-1', description: 'AWS region')
    string(name: 'CLUSTER_NAME', defaultValue: 'my-cluster',  description: 'EKS cluster name')
    string(name: 'K8S_DIR',      defaultValue: 'k8s',         description: 'Path to Kubernetes manifests')
    string(name: 'NAMESPACE',    defaultValue: 'default',     description: 'Kubernetes namespace')
  }

  environment {
    DOCKER_IMAGE = 'akashvb/trend-static-app:latest'
    AWS_DEFAULT_REGION = "${params.AWS_REGION}"
  }

  options {
    timestamps()
    ansiColor('xterm')
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build Docker Image') {
      steps {
        sh "docker build -t ${DOCKER_IMAGE} ."
      }
    }

    stage('Push to DockerHub') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'docker-hub',
          usernameVariable: 'DOCKER_USER',
          passwordVariable: 'DOCKER_PASS'
        )]) {
          sh '''
            set -e
            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
            docker push ${DOCKER_IMAGE}
            docker logout || true
          '''
        }
      }
    }

    stage('Install CLIs if missing (aws/kubectl)') {
      steps {
        sh '''
          set -e
          if ! command -v aws >/dev/null 2>&1; then
            echo "[+] Installing AWS CLI..."
            curl -sL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
            unzip -q awscliv2.zip
            sudo ./aws/install || true
          fi
          if ! command -v kubectl >/dev/null 2>&1; then
            echo "[+] Installing kubectl..."
            curl -sL -o kubectl "https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl"
            chmod +x kubectl && sudo mv kubectl /usr/local/bin/
          fi
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        // If your Jenkins agent uses an EC2 Instance Profile with EKS permissions, you can remove withCredentials.
        withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
          sh """
            set -e
            echo "[+] Update kubeconfig for ${params.CLUSTER_NAME}"
            aws eks update-kubeconfig --name ${params.CLUSTER_NAME} --region ${params.AWS_REGION}

            echo "[+] Create namespace if missing"
            kubectl get ns ${params.NAMESPACE} >/dev/null 2>&1 || kubectl create ns ${params.NAMESPACE}

            echo "[+] Apply manifests from ${params.K8S_DIR}/ to namespace ${params.NAMESPACE}"
            kubectl apply -n ${params.NAMESPACE} -f ${params.K8S_DIR}/

            echo "[+] Show rollout status (best effort)"
            kubectl -n ${params.NAMESPACE} get deploy
            for d in \$(kubectl -n ${params.NAMESPACE} get deploy -o name); do
              kubectl -n ${params.NAMESPACE} rollout status "\$d" || true
            done

            echo "[+] Services & External IPs"
            kubectl -n ${params.NAMESPACE} get svc -o wide
            echo "[+] LB hostnames (if any):"
            kubectl -n ${params.NAMESPACE} get svc -o jsonpath='{range .items[*]}{.metadata.name}{\"\\t\"}{.status.loadBalancer.ingress[*].hostname}{\"\\n\"}{end}' || true
            echo
          """
        }
      }
    }
  }

  post {
    always {
      sh 'docker image ls | head -n 10 || true'
    }
  }
}
