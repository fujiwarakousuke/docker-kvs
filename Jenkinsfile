pipeline {
  agent any
  options { timestamps() }  // 色付けは使わず、時刻だけ付与

  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST = "192.168.10.64"   // 接続先ホスト（ユーザーは Credentials から供給）
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

          # docker-compose v2 単体バイナリ
          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -fsSL -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          # known_hosts 登録（初回のみ）
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
            # リモート疎通と Docker サーバ版確認
            ssh -i "$SSH_KEY" -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes \
              "${SSH_USER}@${BUILD_HOST}" 'echo OK && whoami && id -nG && docker version --format "{{.Server.Version}}"'
          '''
        }
      }
    }

    stage('DockerHub login') {
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
            export PATH="$HOME/.local/bin:$PATH"

            # compose / docker が使う SSH と DOCKER_HOST は "同じシェル内" で設定＆利用
            export DOCKER_SSH_COMMAND="ssh -i $SSH_KEY -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # パスワードは -x 抑止中に投入（Jenkins が値自体はマスクします）
            set +x
            printf "%s" "$DPASS" | docker login -u "$DUSER" --password-stdin
            set -x

            # 参考: コントローラ側の認証ファイル（CLI が DOCKER_HOST=ssh でもこれを使う）
            test -f "$HOME/.docker/config.json" && grep -H "auths" "$HOME/.docker/config.json" || true
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
            export DOCKER_SSH_COMMAND="ssh -i $SSH_KEY -o IdentitiesOnly=yes -o BatchMode=yes -o StrictHostKeyChecking=yes"
            export DOCKER_HOST="ssh://${SSH_USER}@${BUILD_HOST}"

            # Compose ファイル確認
            test -f docker-compose.build.yml && cat docker-compose.build.yml

            # 旧リソース掃除（存在しなくてもOK）
            docker-compose -f docker-compose.build.yml down --remove-orphans || true
            docker volume prune -f || true

            # ビルド → 起動 → 状態確認
            docker-compose -f docker-compose.build.yml build
            docker-compose -f docker-compose.build.yml up -d
            docker-compose -f docker-compose.build.yml ps
          '''
        }
      }
    }
  }
}

