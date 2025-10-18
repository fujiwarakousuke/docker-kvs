pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST = "192.168.10.64"   // SSHのユーザー名はCredentialsから
    PROD_HOST  = "192.168.10.64"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }

  stages {
    stage('Prepare docker-compose & known_hosts') {
      steps {
        sh '''
          set -eux
          DEST="$HOME/.local/bin"
          mkdir -p "$DEST"
          export PATH="$DEST:$PATH"

          # docker-compose v2 (standalone)
          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -fsSL -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          # known_hosts 登録
          mkdir -p "$HOME/.ssh"
          if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
            ssh-keyscan -H "$BUILD_HOST" >> "$HOME/.ssh/known_hosts" || true
          fi
        '''
      }
    }

    stage('Sanity check (SSH & Docker)') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh_build_host',
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER')
        ]) {
          sh '''
            set -eux
            # 直接SSHで基本確認
            ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" 'echo OK && whoami && id -nG && docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Write Docker auth (no daemon / no SSH)') {
      steps {
        withCredentials([
          usernamePassword(credentialsId: 'dockerhub_fuji',
                           usernameVariable: 'DUSER',
                           passwordVariable: 'DPASS')
        ]) {
          sh '''
            set -eux
            mkdir -p "$HOME/.docker"
            umask 077

            # Base64（GNU/BusyBox両対応）
            AUTH="$( (printf "%s" "$DUSER:$DPASS" | base64 -w0) 2>/dev/null || printf "%s" "$DUSER:$DPASS" | base64 )"

            # 最小の config.json を生成（上書き）
            cat > "$HOME/.docker/config.json" <<JSON
{
  "auths": {
    "https://index.docker.io/v1/": { "auth": "${AUTH}" }
  }
}
JSON

            # 生成確認（値はマスクされるが構造だけ出す）
            grep -H "auths" "$HOME/.docker/config.json"
          '''
        }
      }
    }

    stage('Build & Up (compose over SSH)') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh_build_host',
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER')
        ]) {
          sh '''
            set -eux
            export PATH="$HOME/.local/bin:$PATH"

            # --- ssh-agent を起動し鍵をロード（docker/compose が -i を付けなくても通る）---
            eval "$(ssh-agent -s)"
            # 権限を絞ってからロード（厳格環境向け）
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            # DOCKER_HOST は ssh を使用（ssh-agent を使うので DOCKER_SSH_COMMAND は不要）
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # Compose ファイル確認
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 掃除（存在しなくてもOK）
            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            docker volume prune -f || true

            # ビルド → 起動 → 状態確認
            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps

            # 終了時にagentを確実に停止
            ssh-agent -k
          '''
        }
      }
    }
  }
}

