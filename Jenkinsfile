pipeline {
  agent any

  environment {
    AWS_REGION = "us-east-1"
    ECR_REPO_URI = "957444160326.dkr.ecr.us-east-1.amazonaws.com/devops-eks-demo"
    APP_NAME = "devops-eks-demo"
    K8S_NAMESPACE = "default"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Verify AWS Identity (EC2 Role)') {
      steps {
        sh '''
          set -e
          aws sts get-caller-identity
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -e
          IMAGE_TAG=${GIT_COMMIT}
          echo "IMAGE_TAG=${IMAGE_TAG}"
          docker build -t ${APP_NAME}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Login to ECR') {
      steps {
        sh '''
          set -e
          aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO_URI}
        '''
      }
    }

    stage('Tag & Push Image to ECR') {
      steps {
        sh '''
          set -e
          IMAGE_TAG=${GIT_COMMIT}
          docker tag ${APP_NAME}:${IMAGE_TAG} ${ECR_REPO_URI}:${IMAGE_TAG}
          docker push ${ECR_REPO_URI}:${IMAGE_TAG}
          echo "${ECR_REPO_URI}:${IMAGE_TAG}" > image.txt
        '''
      }
    }

    stage('Deploy to EKS') {
      steps {
        sh '''
          set -e
          IMAGE_FULL=$(cat image.txt)
          echo "Deploying image: ${IMAGE_FULL}"

          # Ensure kubeconfig exists for Jenkins user
          aws eks update-kubeconfig --name devops-eks-demo --region ${AWS_REGION}

          # Replace image in deployment.yaml
          sed -i "s|ECR_IMAGE_URI:TAG|${IMAGE_FULL}|g" k8s/deployment.yaml

          # Apply manifests
          kubectl apply -n ${K8S_NAMESPACE} -f k8s/deployment.yaml
          kubectl apply -n ${K8S_NAMESPACE} -f k8s/service.yaml
          kubectl apply -n ${K8S_NAMESPACE} -f k8s/ingress.yaml

          # Show rollout status (may take time once ingress creates ALB)
          kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=120s

          # Show pods placement
          kubectl get pods -n ${K8S_NAMESPACE} -o wide
        '''
      }
    }
  }

  post {
    always {
      sh '''
        set +e
        echo "Docker images on agent:"
        docker images | head -n 20
      '''
    }
  }
}
