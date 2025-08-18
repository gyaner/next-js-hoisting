pipeline {
  agent any

  environment {
    REPO_URL     = 'https://github.com/gyaner/next-js-hoisting'
    EC2_USER     = 'ubuntu'
    EC2_HOST     = '13.127.235.27'
    APP_DIR      = '/opt/nextjs-app'
    DOCKER_IMAGE = 'nextjs-app'
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Deploy to EC2 (build & run in Docker)') {
      steps {
        sshagent(['ec2-ssh-key']) {   // 🔑 Jenkins credential ID
          sh """
            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST '
              set -e

              # Ensure dependencies
              command -v git >/dev/null || sudo apt-get update -y
              command -v git >/dev/null || sudo apt-get install -y git -y
              command -v docker >/dev/null || sudo apt-get install -y docker.io -y

              # Clone or update repo branch on EC2
              if [ -d "$APP_DIR/.git" ]; then
                cd "$APP_DIR"
                git fetch origin "$BRANCH_NAME"
                git checkout -f "$BRANCH_NAME"
                git reset --hard "origin/$BRANCH_NAME"
              else
                sudo mkdir -p "$APP_DIR"
                sudo chown $EC2_USER:$EC2_USER "$APP_DIR"
                git clone --branch "$BRANCH_NAME" "$REPO_URL" "$APP_DIR"
                cd "$APP_DIR"
              fi

              # Build & run container
              SHORT_COMMIT=\$(echo "$GIT_COMMIT" | cut -c1-7)
              IMAGE_TAG="$DOCKER_IMAGE:\$SHORT_COMMIT"
              CONTAINER_NAME="nextjs-\$BRANCH_NAME"

              docker build -t "\$IMAGE_TAG" .
              docker stop "\$CONTAINER_NAME" || true
              docker rm "\$CONTAINER_NAME" || true
              docker run -d -p 3000:3000 --name "\$CONTAINER_NAME" "\$IMAGE_TAG"
            '
          """
        }
      }
    }
  }
}
