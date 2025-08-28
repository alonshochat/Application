pipeline {
  agent any
  options { ansiColor('xterm'); timestamps() }

  environment {
    APP_NAME   = 'calculator-app-alonshochat'   // ECR repo name
    AWS_REGION = 'us-east-1'                    // your region
    PROD_HOST  = credentials('prod-ec2-host')   // Jenkins Secret Text (Prod DNS/IP)
    SSH_KEY_ID = 'prod-ec2-ssh-key'             // Jenkins SSH key (ec2-user)
  }

  stages {
    stage('Checkout') {
      steps { checkout scm }
    }

    // ===== PR (CI) =====
    stage('PR: Build Image') {
      when { changeRequest() }
      steps {
        sh '''
          set -eux
          AWS_ACCOUNT_ID=$(docker run --rm --network host amazon/aws-cli \
            sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          IMAGE="$ECR/$APP_NAME:pr-${CHANGE_ID}-${BUILD_NUMBER}"

          # ensure repo exists (idempotent)
          docker run --rm --network host amazon/aws-cli ecr describe-repositories \
            --repository-names "$APP_NAME" --region "$AWS_REGION" || \
          docker run --rm --network host amazon/aws-cli ecr create-repository \
            --repository-name "$APP_NAME" --region "$AWS_REGION"

          # ECR login password
          PASS=$(docker run --rm --network host amazon/aws-cli \
            ecr get-login-password --region "$AWS_REGION")

          # Build with explicit Dockerfile and context mounted at /w
          docker run --rm --network host \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v "$WORKSPACE":/w -w /w \
            docker:26-cli sh -lc "\
              echo \\"$PASS\\" | docker login --username AWS --password-stdin $ECR && \
              docker build -f /w/Dockerfile -t '$IMAGE' /w"

          echo "$IMAGE" > image_ref.txt
        '''
      }
      post { success { archiveArtifacts artifacts: 'image_ref.txt', fingerprint: true } }
    }

    stage('PR: Test') {
      when { changeRequest() }
      steps {
        sh '''
          set -eu
          docker run --rm --network host \
            -v "$WORKSPACE":/app -w /app \
            python:3.12-slim sh -lc "
              python -m pip install -r requirements.txt &&
              python -m unittest discover -v -s tests -p 'test_*.py' > test.log 2>&1;
              ec=$?; cat test.log; exit $ec
            "
        '''
      }
      post { always { archiveArtifacts artifacts: 'test.log', onlyIfSuccessful: false } }
    }

    stage('PR: Push to ECR') {
      when { changeRequest() }
      steps {
        sh '''
          set -eux
          AWS_ACCOUNT_ID=$(docker run --rm --network host amazon/aws-cli \
            sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          IMAGE=$(cat image_ref.txt)

          PASS=$(docker run --rm --network host amazon/aws-cli \
            ecr get-login-password --region "$AWS_REGION")

          docker run --rm --network host \
            -v /var/run/docker.sock:/var/run/docker.sock \
            docker:26-cli sh -lc "echo \\"$PASS\\" | docker login --username AWS --password-stdin $ECR && docker push '$IMAGE'"
        '''
      }
    }

    // ===== Merge (CD) =====
    stage('CD: Build Image') {
      when { anyOf { branch 'master'; branch 'main' } }
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

          docker run --rm --network host amazon/aws-cli ecr describe-repositories \
            --repository-names "$APP_NAME" --region "$AWS_REGION" || \
          docker run --rm --network host amazon/aws-cli ecr create-repository \
            --repository-name "$APP_NAME" --region "$AWS_REGION"

          PASS=$(docker run --rm --network host amazon/aws-cli \
            ecr get-login-password --region "$AWS_REGION")

          docker run --rm --network host \
            -v /var/run/docker.sock:/var/run/docker.sock \
            -v "$WORKSPACE":/w -w /w \
            docker:26-cli sh -lc "\
              echo \\"$PASS\\" | docker login --username AWS --password-stdin $ECR && \
              docker build -f /w/Dockerfile -t '$IMAGE_CAND' /w && \
              docker tag '$IMAGE_CAND' '$IMAGE_LATEST'"

          printf "%s\n%s\n" "$IMAGE_CAND" "$IMAGE_LATEST" > images.txt
        '''
      }
      post { success { archiveArtifacts artifacts: 'images.txt', fingerprint: true } }
    }

    stage('CD: Test') {
      when { anyOf { branch 'master'; branch 'main' } }
      steps {
        sh '''
          set -eu
          docker run --rm --network host \
            -v "$WORKSPACE":/app -w /app \
            python:3.12-slim sh -lc "
              python -m pip install -r requirements.txt &&
              python -m unittest discover -v -s tests -p 'test_*.py' > test.log 2>&1;
              ec=$?; cat test.log; exit $ec
            "
        '''
      }
      post { always { archiveArtifacts artifacts: 'test.log', onlyIfSuccessful: false } }
    }

    stage('CD: Push to ECR') {
      when { anyOf { branch 'master'; branch 'main' } }
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
              docker:26-cli sh -lc "echo \\"$PASS\\" | docker login --username AWS --password-stdin $ECR && docker push '$IMG'"
          done < images.txt
        '''
      }
    }

    stage('CD: Deploy to Prod EC2') {
      when { anyOf { branch 'master'; branch 'main' } }
      steps {
        sshagent (credentials: ["${SSH_KEY_ID}"]) {
          sh '''
            set -eux
            command -v ssh >/dev/null || {
              if command -v apt-get >/dev/null; then apt-get update && apt-get install -y openssh-client;
              elif command -v yum >/dev/null; then yum install -y openssh-clients;
              elif command -v dnf >/dev/null; then dnf install -y openssh-clients || dnf install -y openssh;
              elif command -v apk >/dev/null; then apk add --no-cache openssh-client; fi
            }

            IMAGE=$(head -n1 images.txt)

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

