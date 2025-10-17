pipeline {
  agent any
  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST = "root@192.168.10.30"   // ← .64 から .30 に変更
    PROD_HOST  = "root@192.168.10.30"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }
  stages {
    stage('Prepare docker-compose & SSH known_hosts') {
      steps {
        // 可能なら credentialsId を入れて使う:
        // sshagent(credentials: ['ssh_build_host']) {
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

          # .30 のホスト鍵を登録（初回の Host key verification 対策）
          mkdir -p ~/.ssh
          ssh-keyscan -H 192.168.10.30 >> ~/.ssh/known_hosts || true

          # 鍵認証の疎通チェック（鍵が無いと失敗します）
          # ssh -o BatchMode=yes "$BUILD_HOST" 'docker version --format="{{.Server.Version}}"'
        '''
        // }
      }
    }

    stage('Build') {
      steps {
        // sshagent(credentials: ['ssh_build_host']) {
        sh '''
          set -eux
          export PATH="$HOME/.local/bin:$PATH"
          export DOCKER_HOST=ssh://${BUILD_HOST}

          # SSH の厳格検証（known_hosts 済み前提）。初回自動許可にしたいなら accept-new に変更可:
          export DOCKER_SSH_COMMAND='ssh -o BatchMode=yes -o StrictHostKeyChecking=yes'

          test -f docker-compose.build.yml && cat docker-compose.build.yml

          docker-compose -f docker-compose.build.yml down || true
          docker volume prune -f || true

          docker-compose -f docker-compose.build.yml build
          docker-compose -f docker-compose.build.yml up -d
          docker-compose -f docker-compose.build.yml ps
        '''
        // }
      }
    }
  }
}

