pipeline {
  agent none
  options { ansiColor('xterm'); timestamps() }

  environment {
    APP_NAME   = 'calculator-app-alon'
    AWS_REGION = 'eu-central-1'
    PROD_HOST  = credentials('prod-ec2-host')
    SSH_KEY_ID = 'prod-ec2-ssh-key'
  }

  stages {
    stage('Checkout') {
      agent { docker { image 'alpine/git' args '--network host' } }
      steps { checkout scm }
    }

    /* ===== PR (CI) ===== */
    stage('PR: Build Image') {
      when { changeRequest() }
      agent {
        docker {
          image 'docker:26-cli'
          args '--user root -v /var/run/docker.sock:/var/run/docker.sock --network host'
        }
      }
      steps {
        sh '''
          set -eux
          apk add --no-cache py3-pip curl jq
          pip install --no-cache-dir awscli
          AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          IMAGE="$ECR/$APP_NAME:pr-${CHANGE_ID}-${BUILD_NUMBER}"
          aws ecr describe-repositories --repository-names "$APP_NAME" --region "$AWS_REGION" || \
            aws ecr create-repository --repository-name "$APP_NAME" --region "$AWS_REGION"
          aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR"
          docker build -t "$IMAGE" .
          echo "$IMAGE" > image_ref.txt
        '''
      }
      post { success { archiveArtifacts artifacts: 'image_ref.txt', fingerprint: true } }
    }

    stage('PR: Test') {
      when { changeRequest() }
      agent { docker { image 'python:3.12-slim' args '--user root' } }
      steps {
        sh 'pip install -r requirements.txt && python -m unittest discover -v -s tests -p "test_*.py" | tee test.log'
      }
      post { always { archiveArtifacts artifacts: 'test.log', onlyIfSuccessful: false } }
    }

    stage('PR: Push to ECR') {
      when { changeRequest() }
      agent {
        docker {
          image 'docker:26-cli'
          args '--user root -v /var/run/docker.sock:/var/run/docker.sock --network host'
        }
      }
      steps {
        sh '''
          set -eux
          apk add --no-cache py3-pip
          pip install --no-cache-dir awscli
          AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          IMAGE=$(cat image_ref.txt)
          aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR"
          docker push "$IMAGE"
        '''
      }
    }

    /* ===== Merge (CD) ===== */
    stage('CD: Build Image') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent {
        docker {
          image 'docker:26-cli'
          args '--user root -v /var/run/docker.sock:/var/run/docker.sock --network host'
        }
      }
      steps {
        sh '''
          set -eux
          apk add --no-cache py3-pip
          pip install --no-cache-dir awscli
          AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          SHORT_SHA=$(echo "$GIT_COMMIT" | cut -c1-7 || true)
          [ -n "$SHORT_SHA" ] || SHORT_SHA=$(git rev-parse --short HEAD)
          IMAGE_CAND="$ECR/$APP_NAME:commit-$SHORT_SHA"
          IMAGE_LATEST="$ECR/$APP_NAME:latest"
          aws ecr describe-repositories --repository-names "$APP_NAME" --region "$AWS_REGION" || \
            aws ecr create-repository --repository-name "$APP_NAME" --region "$AWS_REGION"
          aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR"
          docker build -t "$IMAGE_CAND" .
          docker tag "$IMAGE_CAND" "$IMAGE_LATEST"
          printf "%s\n%s\n" "$IMAGE_CAND" "$IMAGE_LATEST" > images.txt
        '''
      }
      post { success { archiveArtifacts artifacts: 'images.txt', fingerprint: true } }
    }

    stage('CD: Test') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { docker { image 'python:3.12-slim' args '--user root' } }
      steps {
        sh 'pip install -r requirements.txt && python -m unittest discover -v -s tests -p "test_*.py" | tee test.log'
      }
      post { always { archiveArtifacts artifacts: 'test.log', onlyIfSuccessful: false } }
    }

    stage('CD: Push to ECR') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent {
        docker {
          image 'docker:26-cli'
          args '--user root -v /var/run/docker.sock:/var/run/docker.sock --network host'
        }
      }
      steps {
        sh '''
          set -eux
          apk add --no-cache py3-pip
          pip install --no-cache-dir awscli
          AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region "$AWS_REGION")
          ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
          while read -r IMG; do
            [ -n "$IMG" ] || continue
            aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$ECR"
            docker push "$IMG"
          done < images.txt
        '''
      }
    }

    stage('CD: Deploy to Prod EC2') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { docker { image 'alpine:3.20' args '--user root --network host' } }
      steps {
        sshagent (credentials: ["${SSH_KEY_ID}"]) {
          sh '''
            set -eux
            apk add --no-cache openssh-client py3-pip curl jq
            pip install --no-cache-dir awscli
            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text --region "$AWS_REGION")
            ECR="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
            IMAGE=$(head -n1 images.txt)
            ssh -o StrictHostKeyChecking=no ec2-user@"$PROD_HOST" "\
              aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR && \
              docker pull $IMAGE && docker rm -f calc || true && \
              docker run -d --name calc -p 5000:5000 --restart=always $IMAGE"
          '''
        }
      }
    }

    stage('CD: Health Check') {
      when { anyOf { branch 'master'; branch 'main' } }
      agent { docker { image 'curlimages/curl:8.8.0' } }
      steps {
        sh '''
          set -eux
          for i in $(seq 1 12); do
            if curl -sf "http://$PROD_HOST:5000/health" | grep -qi "ok"; then exit 0; fi
            echo "Waiting for health... ($i/12)"; sleep 5
          done
          exit 1
        '''
      }
    }
  }
}

