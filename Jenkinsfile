pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    // ユーザー名は Credentials 側で渡す想定（ここはIP/ホスト名のみ）
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
          # ユーザー権限で docker-compose（単体バイナリ）を用意
          DEST="$HOME/.local/bin"
          mkdir -p "$DEST"
          export PATH="$DEST:$PATH"
          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -L -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          # known_hosts 登録（初回のみ追加）
          mkdir -p ~/.ssh
          if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
            ssh-keyscan -H "$BUILD_HOST" >> ~/.ssh/known_hosts || true
          fi
        '''
      }
    }

    stage('Pre Check') {
      steps {
        sh '''
          set -eux
          # Docker Hub 認証設定があれば軽く表示（無ければスキップ）
          test -f ~/.docker/config.json && grep -H "docker.io" ~/.docker/config.json || true
        '''
      }
    }

    stage('Sanity check (SSH to remote Docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            # 鍵で SSH 疎通 & Docker サーバ版確認（失敗で即中断）
            ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "$SSH_USER@$BUILD_HOST" 'echo OK; whoami; id -nG; docker version --format "{{.Server.Version}}"'
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

            # Compose ファイル確認（無ければ失敗）
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # docker / compose が内部で使う SSH コマンド（秘密鍵・非対話・厳格チェック）
            export DOCKER_SSH_COMMAND="ssh -i $SSH_KEY -o BatchMode=yes -o StrictHostKeyChecking=yes"

            # DOCKER_HOST を SSH で指定（ユーザーは Credentials の SSH_USER を使用）
            export DOCKER_HOST=ssh://$SSH_USER@$BUILD_HOST

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

