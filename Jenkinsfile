pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER           = 'fujiwarakousuke'
    BUILD_HOST               = '192.168.10.64'      // Dockerリモートホスト
    DOCKER_BUILDKIT          = '1'
    COMPOSE_DOCKER_CLI_BUILD = '1'
    DOCKER_CONFIG            = "${WORKSPACE}/.docker" // このジョブ専用の docker 認証保存先
    COMPOSE_PROJECT_NAME     = 'docker-kvs'         // 掃除の安全性のため固定プロジェクト名
    REGISTRY                 = 'docker.io'
    PULL_PLATFORM            = 'linux/amd64'        // 例：x86_64ホスト。ARMなら linux/arm64 に変更
  }

  stages {

    stage('Prepare compose & known_hosts') {
      steps {
        sh '''
          set -eux
          DEST="$HOME/.local/bin"
          mkdir -p "$DEST" "$DOCKER_CONFIG" "$HOME/.ssh"
          export PATH="$DEST:$PATH"

          # docker-compose v2 単体バイナリ（クライアント側）
          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -fsSL -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          # known_hosts へリモート鍵登録
          if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
            ssh-keyscan -H "$BUILD_HOST" >> "$HOME/.ssh/known_hosts"
          fi
          chmod 700 "$HOME/.ssh"
          chmod 600 "$HOME/.ssh/known_hosts"
        '''
      }
    }

    stage('Sanity check (SSH & remote docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            chmod 600 "$SSH_KEY" || true
            ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" \
              'echo OK && whoami && id -nG && docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Login to Docker Hub (local & remote)') {
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

            # クライアント側に認証保存（daemon不要）
            mkdir -p "$DOCKER_CONFIG"
            echo "$DPASS" | docker login -u "$DUSER" --password-stdin "$REGISTRY"

            # リモート側にも明示ログイン（レート制限/401対策）
            ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" \
              "mkdir -p ~/.docker && echo '$DPASS' | docker login -u '$DUSER' --password-stdin '$REGISTRY'"
          '''
        }
      }
    }

    stage('Tune remote daemon (best effort)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" '
              set -e
              if command -v sudo >/dev/null 2>&1 && sudo -n true 2>/dev/null; then
                TMP=$(mktemp)
                if [ -f /etc/docker/daemon.json ]; then
                  sudo cp /etc/docker/daemon.json "$TMP"
                else
                  echo "{}" > "$TMP"
                fi
                python3 - "$TMP" <<PY || true
import json, sys
p=sys.argv[1]
j=json.load(open(p))
j["max-concurrent-downloads"]=1
open(p,"w").write(json.dumps(j))
PY
                sudo mv "$TMP" /etc/docker/daemon.json
                sudo systemctl restart docker
                echo "[remote] docker daemon tuned: max-concurrent-downloads=1"
              else
                echo "[warn] passwordless sudo not available; skip daemon tuning"
              fi
            '
          '''
        }
      }
    }

    stage('Build & Up on remote (pull with retry)') {
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

            # ===== ssh-agent 準備 =====
            eval "$(ssh-agent -s)"
            trap 'ssh-agent -k || true' EXIT
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY" >/dev/null

            export PATH="$HOME/.local/bin:$PATH"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # ===== 補助関数 =====
            refresh_auth() {
              # 使い方: refresh_auth <user> <pass>
              local duser="$1" dpass="$2"
              set +x
              echo "$dpass" | docker login -u "$duser" --password-stdin "$REGISTRY" >/dev/null 2>&1 || true
              set -x
              ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
                "${SSH_USER}@${BUILD_HOST}" \
                "echo '$dpass' | docker login -u '$duser' --password-stdin '$REGISTRY' >/dev/null 2>&1 || true"
            }

            docker_pull_retry_auth() {
              # 使い方: docker_pull_retry_auth [--platform linux/amd64] IMAGE[:TAG]
              local opts=()
              while [ $# -gt 0 ] && echo "$1" | grep -q '^--'; do
                opts="$opts $1 $2"; shift 2
              done
              local img="$1"; shift || true

              local max=7 try=1 rc=0 out=
              while [ $try -le $max ]; do
                echo "[pull] ($try/$max) $opts $img"
                set +e
                out="$(eval docker pull $opts \"$img\" 2>&1)"; rc=$?
                set -e
                echo "$out"
                if [ $rc -eq 0 ]; then
                  return 0
                fi
                if echo "$out" | grep -qiE 'unauthorized|denied|authentication required'; then
                  echo "[pull] auth error; refreshing credentials..."
                  refresh_auth "$DUSER" "$DPASS"
                fi
                sleep $((10 * try))
                try=$((try+1))
              done
              echo "[pull] FAILED: $opts $img"
              return 1
            }

            # ===== 先に必要イメージをタグで取得（digest固定はしない）=====
            docker_pull_retry_auth --platform "$PULL_PLATFORM" "docker.io/selenium/hub:3.141.59-vanadium"
            docker_pull_retry_auth --platform "$PULL_PLATFORM" "docker.io/selenium/node-chrome:3.141.59-vanadium"
            docker_pull_retry_auth --platform "$PULL_PLATFORM" "docker.io/redis:5.0.6-alpine3.10"

            # ===== Compose 実行（リモートエンジン上）=====
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml down --remove-orphans || true
            # プロジェクトに紐づくボリュームも掃除（必要でなければ外してください）
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

