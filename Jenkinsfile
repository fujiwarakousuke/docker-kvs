pipeline {
  agent any
  options { timestamps() }

  environment {
    BUILD_HOST = "192.168.10.64"                     // リモートDockerホスト
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
    DOCKER_CONFIG = "${WORKSPACE}/.docker"           // このジョブ専用の認証保存先(クライアント側)
    LOCALBIN = "${env.WORKSPACE}/bin"
  }

  stages {

    stage('Prepare (compose & ssh wrapper & known_hosts)') {
      steps {
        sh '''
          set -eux
          mkdir -p "$LOCALBIN" "$DOCKER_CONFIG" "$HOME/.ssh"
          export PATH="$LOCALBIN:$PATH"

          # ssh ラッパ（DOCKER_HOST=ssh:// が呼ぶ ssh を上書き）
          cat > "$LOCALBIN/ssh" <<'EOS'
#!/bin/sh
exec /usr/bin/ssh -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes "$@"
EOS
          chmod +x "$LOCALBIN/ssh"

          # docker-compose(単体バイナリ) が無ければ取得
          if ! command -v docker-compose >/dev/null 2>&1; then
            mkdir -p "$HOME/.local/bin"
            curl -fsSL -o "$HOME/.local/bin/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$HOME/.local/bin/docker-compose"
            export PATH="$HOME/.local/bin:$PATH"
          fi
          docker-compose version

          # known_hosts へ BUILD_HOST を登録（初回のみ）
          ssh-keygen -F "${BUILD_HOST}" >/dev/null || ssh-keyscan -H "${BUILD_HOST}" >> "$HOME/.ssh/known_hosts" || true
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
            ssh -i "$SSH_KEY" "${SSH_USER}@${BUILD_HOST}" \
              'echo OK; whoami; id -nG; docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Docker Hub login (CLIENT side)') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub_fuji',
                                          usernameVariable: 'DUSER',
                                          passwordVariable: 'DPASS')]) {
          sh '''
            set -eux
            mkdir -p "$DOCKER_CONFIG"
            echo "$DPASS" | docker login -u "$DUSER" --password-stdin
            test -s "$DOCKER_CONFIG/config.json" && grep -q '"auths"' "$DOCKER_CONFIG/config.json"
          '''
        }
      }
    }

    stage('Build & Up via remote Docker (DOCKER_HOST=ssh://)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            # ssh-agent で鍵を積む（ラッパ ssh から利用）
            eval "$(ssh-agent -s)"
            trap 'ssh-agent -k || true' EXIT
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            export PATH="$LOCALBIN:$HOME/.local/bin:$PATH"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # compose ファイルの存在確認
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # ここまでに client 側は docker login 済み → Pull で認証が効く
            docker pull selenium/hub:3.141.59-vanadium
            docker pull selenium/node-chrome:3.141.59-vanadium
            docker pull redis:5.0.6-alpine3.10

            # 掃除 → ビルド → 起動 → 状態確認
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

