pipeline {
  agent any

  environment {
    // Your repo (SSH or HTTPS)
    REPO_URL = 'https://github.com/gyaner/next-js-hoisting'
    EC2_USER = 'ubuntu'
    EC2_HOST = '65.2.132.236'
    APP_DIR  = '/opt/nextjs-app'          // target folder on EC2
    DOCKER_IMAGE = 'nextjs-app'           // local-only tag on EC2
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build/Test (optional)') {
      steps {
        sh '''
          if [ -f package.json ]; then
            npm ci || npm install
            npm run lint || true
            npm test --if-present || true
          fi
        '''
      }
    }

    stage('Deploy to EC2 (clone & build there)') {
      steps {
        sshagent(['ec2-ssh-key']) { // Jenkins SSH key for EC2
          sh """
            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST '
              set -e

              # Ensure deps
              command -v git >/dev/null || sudo apt-get update -y
              command -v git >/dev/null || sudo apt-get install -y git
              command -v docker >/dev/null || sudo apt-get install -y docker.io

              # Clone or update branch on EC2
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

              # Build & (re)run container
              SHORT_COMMIT=$(echo "$GIT_COMMIT" | cut -c1-7)
              IMAGE_TAG="$DOCKER_IMAGE:$SHORT_COMMIT"
              CONTAINER_NAME="nextjs-$BRANCH_NAME"

              docker build -t "$IMAGE_TAG" .
              docker stop "$CONTAINER_NAME" || true
              docker rm "$CONTAINER_NAME" || true
              docker run -d -p 3000:3000 --name "$CONTAINER_NAME" "$IMAGE_TAG"
            '
          """
        }
      }
    }
  }
}
