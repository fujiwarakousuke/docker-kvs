pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER           = 'fujiwarakousuke'
    BUILD_HOST               = '192.168.10.64'
    COMPOSE_PROJECT_NAME     = 'docker-kvs'
    REGISTRY                 = 'docker.io'
    PULL_PLATFORM            = 'linux/amd64'          // ARMなら linux/arm64
    DOCKER_CONFIG            = "${WORKSPACE}/.docker" // ローカル用
    DOCKER_BUILDKIT          = '1'
    COMPOSE_DOCKER_CLI_BUILD = '1'
  }

  stages {

    stage('Prepare compose & known_hosts') {
      steps {
        sh '''
          set -eux
          DEST="$HOME/.local/bin"
          mkdir -p "$DEST" "$DOCKER_CONFIG" "$HOME/.ssh"
          export PATH="$DEST:$PATH"

          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -fsSL -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version || true

          if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
            ssh-keyscan -H "$BUILD_HOST" >> "$HOME/.ssh/known_hosts"
          fi
          chmod 700 "$HOME/.ssh"; chmod 600 "$HOME/.ssh/known_hosts"
        '''
      }
    }

    stage('Sanity (SSH, remote docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            SSH_OPTS='-o BatchMode=yes -o StrictHostKeyChecking=yes -o ConnectTimeout=15'
            chmod 600 "$SSH_KEY" || true
            ssh $SSH_OPTS -i "$SSH_KEY" "${SSH_USER}@${BUILD_HOST}" 'echo OK && whoami && docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Login (local & remote)') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh_build_host',
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: 'dockerhub_fuji',
                           usernameVariable: 'DUSER',
                           passwordVariable: 'DPASS')
        ]) {
          sh '''
            set -eux
            # ローカル
            mkdir -p "$DOCKER_CONFIG"
            printf '%s\n' "$DPASS" | docker login -u "$DUSER" --password-stdin "$REGISTRY"

            # リモート
            SSH_OPTS='-o BatchMode=yes -o StrictHostKeyChecking=yes -o ConnectTimeout=15'
            ssh $SSH_OPTS -i "$SSH_KEY" "${SSH_USER}@${BUILD_HOST}" \
              "mkdir -p ~/.docker && printf '%s\\n' '$DPASS' | docker login -u '$DUSER' --password-stdin '$REGISTRY'"
          '''
        }
      }
    }

    stage('Pre-pull large base images (via SSH, with retry)') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh_build_host',
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: 'dockerhub_fuji',
                           usernameVariable: 'DUSER',
                           passwordVariable: 'DPASS')
        ]) {
          sh '''
            set -eux
            SSH_BASE="-o BatchMode=yes -o StrictHostKeyChecking=yes -o ServerAliveInterval=15 -o ServerAliveCountMax=8 -o ConnectTimeout=30"
            export SSH_KEY SSH_USER

            refresh_auth_remote() {
              printf '%s\n' "$DPASS" | ssh $SSH_BASE -i "$SSH_KEY" "${SSH_USER}@${BUILD_HOST}" \
                docker login -u "$DUSER" --password-stdin "$REGISTRY" >/dev/null 2>&1 || true
            }

            pull_retry_remote() {
              plat="$1"; img="$2"; max=7; try=1
              while [ "$try" -le "$max" ]; do
                echo "[pull][remote] ($try/$max) --platform $plat $img"
                set +e
                out=$(ssh $SSH_BASE -i "$SSH_KEY" "${SSH_USER}@${BUILD_HOST}" \
                      docker pull --platform "$plat" "$img" 2>&1); rc=$?
                set -e
                echo "$out"
                [ "$rc" -eq 0 ] && return 0

                echo "$out" | grep -qiE 'unauthorized|denied|authentication required' && {
                  echo "[pull] auth error; re-login remote..."
                  refresh_auth_remote
                }

                # ルート不達/タイムアウト系は待って再試行
                echo "$out" | grep -qiE 'No route to host|Timeout, server .* not responding|Connection timed out' && {
                  echo "[pull] network issue; will retry..."
                }

                sleep $((10 * try))
                try=$((try + 1))
              done
              echo "[pull] FAILED (remote) $img"
              return 1
            }

            # ここはタグ指定（digest固定しない）
            pull_retry_remote "${PULL_PLATFORM}" docker.io/selenium/hub:3.141.59-vanadium
            pull_retry_remote "${PULL_PLATFORM}" docker.io/selenium/node-chrome:3.141.59-vanadium
            pull_retry_remote "${PULL_PLATFORM}" docker.io/redis:5.0.6-alpine3.10
          '''
        }
      }
    }

    stage('Compose build & up (use DOCKER_HOST=ssh only here)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            # DOCKER_HOST=ssh 用の ssh コマンド（KeepAlive 付）
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"
            export DOCKER_SSH_CMD="ssh -i ${SSH_KEY} -o BatchMode=yes -o StrictHostKeyChecking=yes -o ServerAliveInterval=15 -o ServerAliveCountMax=8 -o ConnectTimeout=30"
            export PATH="$HOME/.local/bin:$PATH"

            # 接続健全性チェック（短時間で失敗なら明示的に落とす）
            ${DOCKER_SSH_CMD} "${SSH_USER}@${BUILD_HOST}" true

            test -f docker-compose.build.yml && cat docker-compose.build.yml

            docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml down --remove-orphans || true
            # プロジェクトの匿名ボリュームだけ掃除（存在すれば）
            docker volume ls -q --filter "label=com.docker.compose.project=${COMPOSE_PROJECT_NAME}" | xargs -r docker volume rm || true

            docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml build
            docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml up -d
            docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml ps
          '''
        }
      }
    }
  }
}

