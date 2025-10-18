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
    stage('Prepare docker-compose & known_hosts') {
      steps {
        sh '''
          set -eux
          # 念のため過去の環境変数を無効化
          unset DOCKER_SSH_COMMAND || true
          unset GIT_SSH_COMMAND || true
          unset SSH_COMMAND || true

          DEST="$HOME/.local/bin"
          mkdir -p "$DEST"
          export PATH="$DEST:$PATH"

          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -fsSL -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          mkdir -p "$HOME/.ssh"
          if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
            ssh-keyscan -H "$BUILD_HOST" >> "$HOME/.ssh/known_hosts" || true
          fi
        '''
      }
    }

    stage('Sanity check (SSH & Docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            unset DOCKER_SSH_COMMAND || true
            ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" 'echo OK && whoami && id -nG && docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Write Docker auth (local config.json)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub_fuji',
                                          usernameVariable: 'DUSER',
                                          passwordVariable: 'DPASS')]) {
          sh '''
            set -eux
            unset DOCKER_SSH_COMMAND || true
            mkdir -p "$HOME/.docker"
            umask 077
            AUTH="$( (printf "%s" "$DUSER:$DPASS" | base64 -w0) 2>/dev/null || printf "%s" "$DUSER:$DPASS" | base64 )"
            cat > "$HOME/.docker/config.json" <<JSON
{
  "auths": {
    "https://index.docker.io/v1/": { "auth": "${AUTH}" }
  }
}
JSON
            grep -H "auths" "$HOME/.docker/config.json"
          '''
        }
      }
    }

    stage('Build & Up (compose over SSH, via agent)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            export PATH="$HOME/.local/bin:$PATH"

            # 0) 古いDOCKER_SSH_COMMAND/GIT_SSH_COMMANDを確実に殺す
            unset DOCKER_SSH_COMMAND || true
            unset GIT_SSH_COMMAND || true
            unset SSH_COMMAND || true

            # 1) ssh-agent起動 & 鍵ロード
            eval "$(ssh-agent -s)"
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            # 2) ~/.ssh/config を完全に無視するラッパを作成（-i指定なし、agent優先）
            SSH_WRAP="$WORKSPACE/.sshwrap"
            cat > "$SSH_WRAP" <<'SH'
#!/bin/sh
# ignore user ssh config and rely on ssh-agent
exec ssh -F /dev/null -o IdentitiesOnly=no -o BatchMode=yes -o StrictHostKeyChecking=yes "$@"
SH
            chmod +x "$SSH_WRAP"
            export DOCKER_SSH_COMMAND="$SSH_WRAP"

            # 3) リモートdockerに接続するエンドポイント
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # 4) Composeファイル確認
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 5) 掃除（存在しなくてもOK）
            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            docker volume prune -f || true

            # 6) ビルド → 起動 → 状態確認
            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps

            # 7) agent停止
            ssh-agent -k
          '''
        }
      }
    }
  }
}

