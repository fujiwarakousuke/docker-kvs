peline {
  agent any

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    // リモート Docker デーモンが動いているホスト（IPのみ）
    BUILD_HOST = "192.168.10.64"
    PROD_HOST  = "192.168.10.64"
    // BuildKit / compose ビルドを有効化
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

          # known_hosts 登録（初回のみ）
          mkdir -p ~/.ssh
          chmod 700 ~/.ssh
          if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
            ssh-keyscan -H "$BUILD_HOST" >> ~/.ssh/known_hosts || true
          fi
          chmod 600 ~/.ssh/known_hosts
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
        withCredentials([sshUserPrivateKey(
          credentialsId: 'ssh_build_host',
          keyFileVariable: 'SSH_KEY',
          usernameVariable: 'SSH_USER'
        )]) {
          sh '''
            set -eux
            # ssh-agent に鍵を登録（このシェルの間だけ有効）
            eval "$(ssh-agent -s)"
            trap 'ssh-agent -k' EXIT
            ssh-add "$SSH_KEY"

            # 疎通確認：Docker サーバ版を取得
            ssh -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "$SSH_USER@${BUILD_HOST}" \
              'echo OK; whoami; id -nG; docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Build on remote with compose (over SSH)') {
      steps {
        withCredentials([sshUserPrivateKey(
          credentialsId: 'ssh_build_host',
          keyFileVariable: 'SSH_KEY',
          usernameVariable: 'SSH_USER'
        )]) {
          sh '''
            set -eux
            export PATH="$HOME/.local/bin:$PATH"

            # ssh-agent に鍵を登録（compose/ssh が自動で利用）
            eval "$(ssh-agent -s)"
            trap 'ssh-agent -k' EXIT
            ssh-add "$SSH_KEY"

            # Docker は SSH 経由でリモートへ
            export DOCKER_HOST=ssh://$SSH_USER@${BUILD_HOST}

            # Composeファイル確認（ワークスペース直下を想定）
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 旧リソース掃除（存在しなくてもOK）
            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            # 共有環境なら volume prune は **慎重に**（必要ならコメント解除）
            # docker volume prune -f || true

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
pipeline {
  agent any
  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    // ホストは「IPのみ」を入れる（ユーザー名は Credentials から受け取る）
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
            # 鍵でSSH疎通＆Dockerサーバ版取得（失敗したらここで止まる）
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

            # Composeファイル確認（無ければfail）
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # Docker/Compose が使う SSH コマンド（秘密鍵・非対話・厳格チェック）
            export DOCKER_SSH_COMMAND="ssh -i $SSH_KEY -o BatchMode=yes -o StrictHostKeyChecking=yes"

            # DOCKER_HOST を SSH で指定（ユーザーは Credentials の SSH_USER を使用）
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

