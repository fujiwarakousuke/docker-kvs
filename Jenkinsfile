
pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST     = "192.168.10.64"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
    // ここが重要：このジョブ専用の docker 認証ディレクトリ
    DOCKER_CONFIG = "${WORKSPACE}/.docker"
  }

  stages {
    stage('Prepare docker-compose & known_hosts') {
      steps {
        sh '''
          set -eux
          unset DOCKER_SSH_COMMAND GIT_SSH_COMMAND SSH_COMMAND || true

          DEST="$HOME/.local/bin"
          mkdir -p "$DEST" "$DOCKER_CONFIG"
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

    stage('Sanity check (plain SSH & Docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" 'echo OK && whoami && id -nG && docker version --format "{{.Server.Version}}"'
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
            # クライアント側(Jenkins)の DOCKER_CONFIG に資格情報を保存
            mkdir -p "$DOCKER_CONFIG"
            echo "$DPASS" | docker login -u "$DUSER" --password-stdin
            # 最小検証（トークンが入っているか）
            grep -q '"auths"' "$DOCKER_CONFIG/config.json"
          '''
        }
      }
    }

    stage('Build & Up via compose over SSH') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux

            # 1) PATH 先頭に ssh ラッパを配置（/usr/bin/ssh を直叩き）
            mkdir -p "$WORKSPACE/bin"
            cat > "$WORKSPACE/bin/ssh" <<'SH'
#!/bin/sh
exec /usr/bin/ssh -F /dev/null \
  -o IdentitiesOnly=no \
  -o BatchMode=yes \
  -o StrictHostKeyChecking=yes \
  "$@"
SH
            chmod +x "$WORKSPACE/bin/ssh"
            export PATH="$WORKSPACE/bin:$PATH"

            # 2) ssh-agent に鍵を載せる（ラッパは -i を使わない）
            eval "$(ssh-agent -s)"
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            # 3) Docker は上の ssh ラッパを使う。DOCKER_HOST は ssh://
            export PATH="$HOME/.local/bin:$PATH"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # 4) Compose ファイル確認
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 5) 先に pull して認証を確認（失敗すればここで止まる）
            docker pull selenium/hub:3.141.59-vanadium
            docker pull selenium/node-chrome:3.141.59-vanadium
            docker pull redis:5.0.6-alpine3.10

            # 6) 掃除 → ビルド → 起動 → 確認
            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            docker volume prune -f || true

            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps

            # 7) agent 終了
            ssh-agent -k
          '''
        }
      }
    }
  }
}
pipeline {
  agent any
  options { timestamps() }

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST = "192.168.10.64"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }

  stages {
    stage('Prepare docker-compose & known_hosts') {
      steps {
        sh '''
          set -eux
          # 古い設定を無効化
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

    stage('Sanity check (plain SSH & Docker)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux
            # 鍵で素のSSH試験
            ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" 'echo OK && whoami && id -nG && docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('Docker registry login (remote)') {
      steps {
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh_build_host', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER'),
          usernamePassword(credentialsId: 'dockerhub_fuji', usernameVariable: 'DUSER', passwordVariable: 'DPASS')
        ]) {
          sh '''
            set -eux
            # リモート側で docker login（デーモンに資格を入れる）
            ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" "echo '$DPASS' | docker login -u '$DUSER' --password-stdin"
          '''
        }
      }
    }

    stage('Build & Up via compose over SSH (force our ssh)') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ssh_build_host',
                                           keyFileVariable: 'SSH_KEY',
                                           usernameVariable: 'SSH_USER')]) {
          sh '''
            set -eux

            # 0) PATH 先頭に ssh ラッパを配置（/usr/bin/ssh を直叩き）
            mkdir -p "$WORKSPACE/bin"
            cat > "$WORKSPACE/bin/ssh" <<'SH'
#!/bin/sh
exec /usr/bin/ssh -F /dev/null \
  -o IdentitiesOnly=no \
  -o BatchMode=yes \
  -o StrictHostKeyChecking=yes \
  "$@"
SH
            chmod +x "$WORKSPACE/bin/ssh"
            export PATH="$WORKSPACE/bin:$PATH"

            # 1) ssh-agent に鍵を載せる（ラッパは -i を一切使わない）
            eval "$(ssh-agent -s)"
            chmod 600 "$SSH_KEY" || true
            ssh-add "$SSH_KEY"

            # 2) docker は ssh を PATH から探すので、上のラッパが必ず使われる
            export PATH="$HOME/.local/bin:$PATH"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # 3) Composeファイルを確認
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 4) 掃除
            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            docker volume prune -f || true

            # 5) ビルド→起動→確認
            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps

            # 6) agent 終了
            ssh-agent -k
          '''
        }
      }
    }
  }
}

