pipeline {
  agent any
  options { timestamps() }

  environment {
    BUILD_HOST = "192.168.10.64"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
    DOCKER_CONFIG = "${WORKSPACE}/.docker"     // このジョブ専用の認証保存先(クライアント側)
    LOCALBIN = "${env.WORKSPACE}/bin"
    COMPOSE_PROJECT_NAME = "dockerkvs"         // プロジェクト名を固定（ネットワーク/ボリューム衝突回避）
  }

  stages {

    stage('Prepare (compose & known_hosts)') {
      steps {
        sh '''
          set -eux
          mkdir -p "$LOCALBIN" "$DOCKER_CONFIG" "$HOME/.ssh" "$HOME/.docker/cli-plugins"

          # known_hosts 登録（初回のみ）
          ssh-keygen -F "${BUILD_HOST}" >/dev/null || ssh-keyscan -H -T 3 "${BUILD_HOST}" >> "$HOME/.ssh/known_hosts" || true

          # Compose: 可能なら "docker compose" を使う。無ければ v2 単体バイナリを取得して docker-compose で代替。
          if docker compose version >/dev/null 2>&1; then
            echo "docker compose available"
          else
            if ! command -v docker-compose >/dev/null 2>&1; then
              curl -fsSL -o "$HOME/.docker/cli-plugins/docker-compose" \
                "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
              chmod +x "$HOME/.docker/cli-plugins/docker-compose"
            fi
            docker-compose version
          fi
        '''
      }
    }

    stage('Sanity check (SSH & Docker)') {
      steps {
        sshagent(credentials: ['ssh_build_host']) {
          sh '''
            set -eux
            # 接続確認（対話を避ける）
            ssh -o BatchMode=yes -o StrictHostKeyChecking=yes "${SSH_USER}@${BUILD_HOST}" \
              'echo OK && whoami && id -nG && docker version --format "{{.Server.Version}}"'
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

    stage('Build & Up via remote Docker') {
      steps {
        sshagent(credentials: ['ssh_build_host']) {
          sh '''
            set -eux

            # リモート Docker へ SSH で接続させる設定（ssh バイナリを差し替えずにオプション指定）
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"
            export DOCKER_SSH_COMMAND='ssh -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes'

            # Compose コマンド解決
            if docker compose version >/dev/null 2>&1; then
              COMPOSE='docker compose'
            else
              export PATH="$HOME/.docker/cli-plugins:$PATH"
              COMPOSE='docker-compose'
            fi

            # 参照用
            test -f docker-compose.build.yml && sed -n '1,120p' docker-compose.build.yml

            # （任意）事前 Pull。Docker Hub 認証はクライアント側 config.json を使って転送される
            docker pull selenium/hub:3.141.59-vanadium || true
            docker pull selenium/node-chrome:3.141.59-vanadium || true
            docker pull redis:5.0.6-alpine3.10 || true

            # 掃除 → ビルド → 起動
            ${COMPOSE} -f docker-compose.build.yml down -v --remove-orphans || true
            ${COMPOSE} -f docker-compose.build.yml build --pull --progress=plain
            ${COMPOSE} -f docker-compose.build.yml up -d

            ${COMPOSE} -f docker-compose.build.yml ps
          '''
        }
      }
    }
  }
}

