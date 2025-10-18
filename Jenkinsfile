pipeline {
  agent any
  options { timestamps() }

  environment {
    BUILD_HOST = '192.168.10.64'                 // リモート Docker ホスト
    LOCALBIN   = "${WORKSPACE}/bin"              // ssh ラッパ配置先
    DOCKER_CONFIG = "${WORKSPACE}/.docker"       // このジョブ専用の docker 認証ディレクトリ
    DOCKER_BUILDKIT = '1'
    COMPOSE_DOCKER_CLI_BUILD = '1'
  }

  stages {

    stage('Prepare (ssh wrapper & known_hosts)') {
      steps {
        sh '''
          set -eux
          mkdir -p "$LOCALBIN" "$DOCKER_CONFIG" "$HOME/.ssh"

          # ssh ラッパ（DOCKER_HOST=ssh:// が呼び出す ssh を固定）
          cat > "$LOCALBIN/ssh" <<'EOS'
#!/bin/sh
exec /usr/bin/ssh -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes "$@"
EOS
          chmod +x "$LOCALBIN/ssh"

          # known_hosts に BUILD_HOST を登録（初回のみ）
          ssh-keygen -F "${BUILD_HOST}" >/dev/null || \
            ssh-keyscan -H "${BUILD_HOST}" >> "$HOME/.ssh/known_hosts" || true

          # compose コマンド確認
          docker-compose version || docker compose version
        '''
      }
    }

    stage('Sanity (SSH & remote Docker reachable)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            eval "$(ssh-agent -s)"; trap 'ssh-agent -k || true' EXIT
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            "$LOCALBIN/ssh" "${SSH_USER}@${BUILD_HOST}" \
              'echo OK; whoami; id -nG; docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Docker Hub login (client side)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub_fuji',
                                          usernameVariable: 'DUSER',
                                          passwordVariable: 'DPASS')]) {
          sh '''
            set -eux
            mkdir -p "$DOCKER_CONFIG"
            echo "$DPASS" | docker login -u "$DUSER" --password-stdin
            test -s "$DOCKER_CONFIG/config.json"
          '''
        }
      }
    }

    stage('Build & Up on remote (compose over SSH)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux

            # 過去の漏れた設定を無効化
            unset DOCKER_SSH_COMMAND GIT_SSH_COMMAND SSH_COMMAND || true

            # ssh-agent に鍵を積む（以降は -i を使わない）
            eval "$(ssh-agent -s)"; trap 'ssh-agent -k || true' EXIT
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            export PATH="$LOCALBIN:$PATH"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"
            export DOCKER_SSH_COMMAND="$LOCALBIN/ssh"

            # 確認用（任意）
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 公開イメージの pre-pull（失敗しても続行）
            docker pull selenium/hub:3.141.59-vanadium || true
            docker pull selenium/node-chrome:3.141.59-vanadium || true
            docker pull redis:5.0.6-alpine3.10 || true

            # クリーンアップ → ビルド → 起動 → 状態確認
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

