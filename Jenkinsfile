pipeline {
  agent any

  environment {
    AWS_REGION    = 'us-east-1'
    CLUSTER_NAME  = 'demo-eks-cluster'
    EKS_VERSION   = '1.29'        // adjust as needed
    NODEGROUP_NAME = 'ng-default'
    NODE_TYPE      = 't3.small'
    NODE_COUNT     = '2'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Configure AWS CLI & eksctl') {
      steps {
        // Jenkins Credentials:
        // Create a "Username with password" credential in Jenkins with:
        //   ID: aws-jenkins-creds
        //   Username: AWS_ACCESS_KEY_ID
        //   Password: AWS_SECRET_ACCESS_KEY
        withCredentials([
          usernamePassword(
            credentialsId: 'aws-jenkins-creds',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
          )
        ]) {
          sh '''
            set -e

            echo "[SETUP] Verifying AWS CLI installation..."
            if ! command -v aws >/dev/null 2>&1; then
              echo "[SETUP] AWS CLI not found. Installing AWS CLI v2..."
              curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
              unzip -q awscliv2.zip
              sudo ./aws/install || ./aws/install
            fi

            echo "[SETUP] Verifying eksctl installation..."
            if ! command -v eksctl >/dev/null 2>&1; then
              echo "[SETUP] eksctl not found. Installing eksctl..."
              curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" \
                | tar xz -C /tmp
              sudo mv /tmp/eksctl /usr/local/bin/eksctl || mv /tmp/eksctl /usr/local/bin/eksctl
            fi

            echo "[SETUP] aws --version"
            aws --version || true

            echo "[SETUP] eksctl version"
            eksctl version || true

            echo "[SETUP] AWS region: ${AWS_REGION}"
          '''
        }
      }
    }

    stage('Create EKS Cluster') {
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'aws-jenkins-creds',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
          )
        ]) {
          sh '''
            set -e

            echo "[EKS] Creating EKS cluster ${CLUSTER_NAME} in ${AWS_REGION}..."

            eksctl create cluster \
              --name "${CLUSTER_NAME}" \
              --version "${EKS_VERSION}" \
              --region "${AWS_REGION}" \
              --nodegroup-name "${NODEGROUP_NAME}" \
              --node-type "${NODE_TYPE}" \
              --nodes "${NODE_COUNT}" \
              --managed \
              --asg-access \
              --alb-ingress-access \
              --full-ecr-access

            echo "[EKS] Cluster ${CLUSTER_NAME} created."

            echo "[EKS] Verifying nodes..."
            aws eks update-kubeconfig --name "${CLUSTER_NAME}" --region "${AWS_REGION}"
            kubectl get nodes -o wide
          '''
        }
      }
    }
  }

  post {
    success {
      echo "EKS cluster ${env.CLUSTER_NAME} created successfully in ${env.AWS_REGION}."
    }
    failure {
      echo "EKS cluster creation failed. Check the logs above."
    }
  }
}

