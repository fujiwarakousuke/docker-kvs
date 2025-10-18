pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER           = "fujiwarakousuke"
    BUILD_HOST               = "192.168.10.64"        // 接続先ホスト
    DOCKER_BUILDKIT          = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
    DOCKER_CONFIG            = "${WORKSPACE}/.docker" // このジョブ専用の docker 認証保存先（クライアント側）
    COMPOSE_PROJECT_NAME     = "docker-kvs"           // プロジェクト名固定（掃除の安全性）
    REGISTRY                 = "docker.io"
  }

  stages {

    stage('Prepare docker-compose & known_hosts') {
      steps {
        sh '''#!/usr/bin/env bash
set -Eeuo pipefail
DEST="$HOME/.local/bin"
mkdir -p "$DEST" "$DOCKER_CONFIG" "$HOME/.ssh"
export PATH="$DEST:$PATH"

# docker-compose (v2 単体バイナリ) が無ければ取得
if ! command -v docker-compose >/dev/null 2>&1; then
  curl -fsSL -o "$DEST/docker-compose" \
    "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
  chmod +x "$DEST/docker-compose"
fi
docker-compose version

# known_hosts へ登録（取得失敗時はビルド停止）
if ! ssh-keygen -F "$BUILD_HOST" >/dev/null; then
  ssh-keyscan -H "$BUILD_HOST" >> "$HOME/.ssh/known_hosts"
fi
chmod 700 "$HOME/.ssh"
chmod 600 "$HOME/.ssh/known_hosts"
'''
      }
    }

    stage('Sanity check (SSH & Docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''#!/usr/bin/env bash
set -Eeuo pipefail
chmod 600 "$SSH_KEY" || true
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
          sh '''#!/usr/bin/env bash
set -Eeuo pipefail
mkdir -p "$DOCKER_CONFIG"
# これはデーモン不要（レジストリへ直接認証）
echo "$DPASS" | docker login -u "$DUSER" --password-stdin "$REGISTRY"
test -s "$DOCKER_CONFIG/config.json"
grep -q '"auths"' "$DOCKER_CONFIG/config.json"
# デーモンに触らない確認だけにする
docker --version
'''
        }
      }
    }

    stage('Build on remote with compose (over SSH + stable auth)') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh_build_host',
                            keyFileVariable: 'SSH_KEY',
                            usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: 'dockerhub_fuji',
                           usernameVariable: 'DUSER',
                           passwordVariable: 'DPASS')
        ]) {
          sh '''#!/usr/bin/env bash
set -Eeuo pipefail

# ===== ssh-agent 準備（DOCKER_HOST=ssh が agent を利用）=====
eval "$(ssh-agent -s)"
trap 'ssh-agent -k || true' EXIT
chmod 600 "$SSH_KEY" || true
ssh-add "$SSH_KEY" >/dev/null
ssh-add -l || true

export PATH="$HOME/.local/bin:$PATH"
export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

# ===== 使うたびに DOCKER_AUTH_CONFIG を生成（クライアント側から毎APIへ確実に送る）=====
refresh_auth() {
  local duser="$1" dpass="$2"
  set +x
  local AUTH
  AUTH="$(printf '%s' "$duser:$dpass" | base64 | tr -d '\\n')"
  export DOCKER_AUTH_CONFIG="$(cat <<JSON
{"auths":{
  "https://index.docker.io/v1/":{"auth":"$AUTH"},
  "https://registry-1.docker.io/v2/":{"auth":"$AUTH"},
  "docker.io":{"auth":"$AUTH"}
}}
JSON
)"
  set -x
}
refresh_auth "$DUSER" "$DPASS"

# ===== 保険：リモート側にも login =====
ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
  "${SSH_USER}@${BUILD_HOST}" '
    mkdir -p ~/.docker
    docker logout docker.io >/dev/null 2>&1 || true
    docker logout https://index.docker.io/v1/ >/dev/null 2>&1 || true
    docker logout https://registry-1.docker.io/v2/ >/dev/null 2>&1 || true
    rm -f ~/.docker/config.json || true
    echo "'"$DPASS"'" | docker login -u "'"$DUSER"'" --password-stdin "'"$REGISTRY"'"
  '

# ===== pull の堅牢化（401時は再ログインして再試行）=====
docker_pull_retry_auth() {
  local img="$1" max=7 try=1
  while (( try <= max )); do
    echo "[pull] ($try/$max) $img"
    set +e
    local out; out="$(docker pull "$img" 2>&1)"; rc=$?
    set -e
    echo "$out"
    if (( rc == 0 )); then return 0; fi
    if echo "$out" | grep -qiE 'unauthorized|denied|authentication required'; then
      echo "[pull] auth error detected; refreshing credentials and re-login..."
      refresh_auth "$DUSER" "$DPASS"
      echo "$DPASS" | docker login -u "$DUSER" --password-stdin "$REGISTRY" || true
      ssh -i "$SSH_KEY" -o BatchMode=yes -o StrictHostKeyChecking=yes \
        "${SSH_USER}@${BUILD_HOST}" \
        "echo '$DPASS' | docker login -u '$DUSER' --password-stdin '$REGISTRY' || true"
    fi
    sleep $(( 10 * try ))
    (( try++ ))
  done
  echo "[pull] FAILED: $img"; return 1
}

# Compose ファイル確認
test -f docker-compose.build.yml && cat docker-compose.build.yml

# 大きいイメージ向け：堅牢 pull
docker_pull_retry_auth "docker.io/selenium/hub:3.141.59-vanadium"
docker_pull_retry_auth "docker.io/selenium/node-chrome:3.141.59-vanadium"
docker_pull_retry_auth "docker.io/redis:5.0.6-alpine3.10"

# 掃除（このプロジェクトのものに限定）
docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml down --remove-orphans || true
docker volume ls -q --filter "label=com.docker.compose.project=${COMPOSE_PROJECT_NAME}" \
  | xargs -r docker volume rm || true

# ビルド → 起動 → 状態確認
docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml build
docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml up -d
docker-compose -p "$COMPOSE_PROJECT_NAME" -f docker-compose.build.yml ps
'''
        }
      }
    }
  }
}

