pipeline {
  agent any
  options { timestamps() }

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
          def dockerConfigExists = sh(
            script: 'test -f ~/.docker/config.json && echo true || echo false',
            returnStdout: true
          ).trim()
          if (dockerConfigExists == 'true') {
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
            # 鍵を直指定して疎通確認（ここは DOCKER_SSH_COMMAND を使う）
            ssh -i "$SSH_KEY" \
                -o IdentitiesOnly=yes \
                -o BatchMode=yes \
                -o StrictHostKeyChecking=yes \
                "${SSH_USER}@${BUILD_HOST}" 'echo OK; whoami; id -nG; docker version --format "{{.Server.Version}}"'
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

            # ★ Compose v2 は DOCKER_SSH_COMMAND を基本見ないので ~/.ssh/config を用意
            mkdir -p ~/.ssh
            cat > ~/.ssh/config <<EOF
Host ${BUILD_HOST}
  HostName ${BUILD_HOST}
  User ${SSH_USER}
  IdentityFile ${SSH_KEY}
  IdentitiesOnly yes
  StrictHostKeyChecking yes
EOF
            chmod 600 ~/.ssh/config

            # DOCKER_HOST は ssh:// ユーザー@ホスト を指定（ssh は ~/.ssh/config を参照）
            export DOCKER_HOST=ssh://${SSH_USER}@${BUILD_HOST}

            # 旧リソース掃除（存在しなくてもOK）
            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            docker volume prune -f || true

            # ビルド → 起動 → 状態確認
            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps
          '''
        }
      }
    }
  }
}

