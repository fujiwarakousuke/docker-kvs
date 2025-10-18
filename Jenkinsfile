pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST     = "192.168.10.64"      // 接続先ホスト(IPのみ)
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
    DOCKER_CONFIG = "${WORKSPACE}/.docker" // このジョブ専用の docker 認証保存先
  }

  stages {
    stage('Prepare docker-compose & known_hosts') {
      steps {
        sh '''
          set -eux
          DEST="$HOME/.local/bin"
          mkdir -p "$DEST" "$DOCKER_CONFIG"
          export PATH="$DEST:$PATH"

          # docker-compose(単体バイナリ) が無ければ取得
          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -fsSL -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          # known_hosts へ登録（初回のみ）
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
            ssh -i "$SSH_KEY" \
                -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
                "${SSH_USER}@${BUILD_HOST}" \
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

    stage('Build on remote with compose (over SSH)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux

            # ssh-agent に鍵を積む（Compose/DOCKER_HOST が ssh を使う際に agent を利用）
            eval "$(ssh-agent -s)"
            trap 'ssh-agent -k || true' EXIT
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            export PATH="$HOME/.local/bin:$PATH"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # Compose ファイル確認
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 事前 pull（認証/到達性の早期検知）
            docker pull selenium/hub:3.141.59-vanadium
            docker pull selenium/node-chrome:3.141.59-vanadium
            docker pull redis:5.0.6-alpine3.10

            # 掃除 → ビルド → 起動 → 確認
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

