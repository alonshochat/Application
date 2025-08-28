pipeline {
  agent none
  options { ansiColor('xterm'); timestamps() }

  environment {
    APP_NAME   = 'calculator-app-alon'
    AWS_REGION = 'us-east-1'
    PROD_HOST  = credentials('prod-ec2-host')
    SSH_KEY_ID = 'prod-ec2-ssh-key'
  }

  stages {
    stage('Checkout') {
      agent { any }
      steps { checkout scm }
    }

    /* ===== PR (CI) ===== */
    stage('PR: Build Image') {
      when { changeRequest() }
      agent { any }
      steps {
        sh '''
          set -eux
          # Discover account and ECR
          AWS_ACCOUNT_ID=$(docker run --rm --network host amazon/aws-cli \
            sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          IMAGE="$ECR/$APP_NAME:pr-${CHANGE_ID}-${BUILD_NUMBER}"

          # Ensure repo exists (idempotent)
          docker run --rm --network host amazon/aws-cli ecr describe-repositories \
            --repository-names "$APP_NAME" --region "$AWS_REGION" || \
          docker run --rm --network host amazon/aws-cli ecr create-repository \
            --repository-name "$APP_NAME" --region "$AWS_REGION"

          # Build using the host docker daemon via docker:26-cli
          docker run --rm --network host \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v "$PWD":/workspace -w /workspace \
            docker:26-cli sh -lc "docker build -t '$IMAGE' ."

          echo "$IMAGE" > image_ref.txt
        '''
      }
      post { success { archiveArtifacts artifacts: 'image_ref.txt', fingerprint: true } }
    }

    stage('PR: Test') {
      when { changeRequest() }
      agent { any }
      steps {
        sh '''
          set -eux
          # Run tests inside a Python container (no plugins needed)
          docker run --rm --network host \
            -v "$PWD":/app -w /app \
            python:3.12-slim sh -lc \
            "pip install -r requirements.txt && python -m unittest discover -v -s tests -p 'test_*.py' | tee test.log"
        '''
      }
      post { always { archiveArtifacts artifacts: 'test.log', onlyIfSuccessful: false } }
    }

    stage('PR: Push to ECR') {
      when { changeRequest() }
      agent { any }
      steps {
        sh '''
          set -eux
          AWS_ACCOUNT_ID=$(docker run --rm --network host amazon/aws-cli \
            sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          IMAGE=$(cat image_ref.txt)

          # ECR login (get password via aws-cli container, use docker CLI container to push)
          PASS=$(docker run --rm --network host amazon/aws-cli \
            ecr get-login-password --region "$AWS_REGION")
          docker run --rm --network host \
            -v /var/run/docker.sock:/var/run/docker.sock \
            docker:26-cli sh -lc "echo \"$PASS\" | docker login --username AWS --password-stdin $ECR && docker push '$IMAGE'"
        '''
      }
    }

    /* ===== Merge (CD) ===== */
    stage('CD: Build Image') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { any }
      steps {
        sh '''
          set -eux
          AWS_ACCOUNT_ID=$(docker run --rm --network host amazon/aws-cli \
            sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          SHORT_SHA=$(git rev-parse --short HEAD || true)
          [ -n "$SHORT_SHA" ] || SHORT_SHA=$(echo "$GIT_COMMIT" | cut -c1-7)

          IMAGE_CAND="$ECR/$APP_NAME:commit-$SHORT_SHA"
          IMAGE_LATEST="$ECR/$APP_NAME:latest"

          # Ensure repo exists
          docker run --rm --network host amazon/aws-cli ecr describe-repositories \
            --repository-names "$APP_NAME" --region "$AWS_REGION" || \
          docker run --rm --network host amazon/aws-cli ecr create-repository \
            --repository-name "$APP_NAME" --region "$AWS_REGION"

          # Build & tag via docker CLI container
          docker run --rm --network host \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v "$PWD":/workspace -w /workspace \
            docker:26-cli sh -lc "docker build -t '$IMAGE_CAND' . && docker tag '$IMAGE_CAND' '$IMAGE_LATEST'"

          printf "%s\n%s\n" "$IMAGE_CAND" "$IMAGE_LATEST" > images.txt
        '''
      }
      post { success { archiveArtifacts artifacts: 'images.txt', fingerprint: true } }
    }

    stage('CD: Test') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { any }
      steps {
        sh '''
          set -eux
          docker run --rm --network host \
            -v "$PWD":/app -w /app \
            python:3.12-slim sh -lc \
            "pip install -r requirements.txt && python -m unittest discover -v -s tests -p 'test_*.py' | tee test.log"
        '''
      }
      post { always { archiveArtifacts artifacts: 'test.log', onlyIfSuccessful: false } }
    }

    stage('CD: Push to ECR') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { any }
      steps {
        sh '''
          set -eux
          AWS_ACCOUNT_ID=$(docker run --rm --network host amazon/aws-cli \
            sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"

          PASS=$(docker run --rm --network host amazon/aws-cli \
            ecr get-login-password --region "$AWS_REGION")
          while read -r IMG; do
            [ -n "$IMG" ] || continue
            docker run --rm --network host \
              -v /var/run/docker.sock:/var/run/docker.sock \
              docker:26-cli sh -lc "echo \"$PASS\" | docker login --username AWS --password-stdin $ECR && docker push '$IMG'"
          done < images.txt
        '''
      }
    }

    stage('CD: Deploy to Prod EC2') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { any }
      steps {
        sshagent (credentials: ["${SSH_KEY_ID}"]) {
          sh '''
            set -eux
            # Ensure ssh client is present on this node
            if ! command -v ssh >/dev/null 2>&1; then
              if command -v apt-get >/dev/null; then apt-get update && apt-get install -y openssh-client;
              elif command -v yum >/dev/null; then yum install -y openssh-clients;
              elif command -v dnf >/dev/null; then dnf install -y openssh-clients || dnf install -y openssh;
              elif command -v apk >/dev/null; then apk add --no-cache openssh-client;
              fi
            fi

            # Figure out candidate image (first line of images.txt)
            IMAGE=$(head -n1 images.txt)

            # Remote deploy (awscli + docker must be installed on Prod EC2; role provides creds)
            ssh -o StrictHostKeyChecking=no ec2-user@"$PROD_HOST" "\
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $(echo $IMAGE | cut -d/ -f1) && \
              docker pull $IMAGE && \
              docker rm -f calc || true && \
              docker run -d --name calc -p 5000:5000 --restart=always $IMAGE"
          '''
        }
      }
    }

    stage('CD: Health Check') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { any }
      steps {
        sh '''
          set -eux
          for i in $(seq 1 12); do
            if curl -sf "http://$PROD_HOST:5000/health" | grep -qi "ok"; then
              echo "Health OK"; exit 0
            fi
            echo "Waiting for health... ($i/12)"; sleep 5
          done
          echo "Health check failed"; exit 1
        '''
      }
    }
  }
}

