pipeline {
  agent any

  options {
    timestamps()
    // ここがポイント：AnsiColor を wrap で指定
    wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm'])
  }

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST = "192.168.10.64"
    PROD_HOST  = "192.168.10.64"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }

  stages {
    stage('Prepare docker-compose & SSH known_hosts') {
      steps {
        sh '''
          set -eux
          DEST="$HOME/.local/bin"
          mkdir -p "$DEST"
          export PATH="$DEST:$PATH"
          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -L -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          mkdir -p ~/.ssh
          if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
            ssh-keyscan -H "$BUILD_HOST" >> ~/.ssh/known_hosts || true
          fi
        '''
      }
    }

    stage('Pre Check') {
      steps {
        script {
          def hasCfg = sh(script: 'test -f ~/.docker/config.json && echo true || echo false', returnStdout: true).trim()
          if (hasCfg == 'true') {
            sh 'grep -H "docker.io" ~/.docker/config.json || true'
          } else {
            echo "Warning: Docker config not found. Skipping Docker login check."
          }
        }
      }
    }

    stage('Sanity check (SSH to remote Docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" 'docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Build on remote with compose (over SSH)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            export PATH="$HOME/.local/bin:$PATH"

            test -f docker-compose.build.yml && cat docker-compose.build.yml

            export DOCKER_SSH_COMMAND="ssh -i $SSH_KEY -o BatchMode=yes -o StrictHostKeyChecking=yes"
            export DOCKER_HOST=ssh://${SSH_USER}@${BUILD_HOST}

            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            docker volume prune -f || true

            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps
          '''
        }
      }
    }
  }
}

