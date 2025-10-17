pipeline {
  agent any
  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST = "root@192.168.10.64"     // root禁止なら jenkins@192.168.10.64
    PROD_HOST  = "root@192.168.10.64"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }
  stages {
    stage('Prepare docker-compose & known_hosts') {
      steps {
        sshagent (credentials: ['ssh_build_host']) {
          sh '''
            set -eux
            # docker-compose 単体バイナリをユーザー権限で配置
            DEST="$HOME/.local/bin"
            mkdir -p "$DEST"
            export PATH="$DEST:$PATH"
            if ! command -v docker-compose >/dev/null 2>&1; then
              curl -L -o "$DEST/docker-compose" \
                "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
              chmod +x "$DEST/docker-compose"
            fi
            docker-compose version

            # known_hosts 登録（初回 handshake 失敗回避）
            mkdir -p ~/.ssh
            ssh-keyscan -H 192.168.10.64 >> ~/.ssh/known_hosts || true

            # SSH 経由で Docker に触れる疎通確認（失敗なら明示で落とす）
            ssh -o BatchMode=yes "$BUILD_HOST" 'docker version --format="{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Pre Check') {
      steps {
        script {
          def hasCfg = sh(script: 'test -f ~/.docker/config.json && echo yes || echo no', returnStdout: true).trim()
          if (hasCfg == 'yes') {
            sh 'grep -H "docker.io" ~/.docker/config.json || true'
          } else {
            echo "Warning: Docker config not found. Skipping Docker login check."
          }
        }
      }
    }

    stage('Build on remote with compose') {
      steps {
        sshagent (credentials: ['ssh_build_host']) {
          sh '''
            set -eux
            export PATH="$HOME/.local/bin:$PATH"
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # DOCKER_HOST をSSHで指定（パスワード非対話のため鍵が必須）
            export DOCKER_HOST=ssh://${BUILD_HOST}

            # SSHオプション（バッチ・KnownHostsを強制）
            export DOCKER_SSH_COMMAND='ssh -o BatchMode=yes -o StrictHostKeyChecking=yes'

            # 旧リソース掃除（存在しなくてもOK）
            docker-compose -f docker-compose.build.yml down || true
            docker volume prune -f || true

            # ビルド→起動→状態確認
            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps
          '''
        }
      }
    }
  }
}

