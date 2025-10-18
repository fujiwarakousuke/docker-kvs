pipeline {
  agent any
  environment {
    DOCKERHUB_USER = "fujiwarakousuke"
    BUILD_HOST = "root@192.168.10.64"
    PROD_HOST  = "root@192.168.10.64"
    DOCKER_BUILDKIT = "1"
    COMPOSE_DOCKER_CLI_BUILD = "1"
  }
  stages {
    stage('Prepare docker-compose & SSH known_hosts') {
      steps {
        sh '''
          set -eux
          # ユーザー権限で置ける場所に docker-compose を配置
          DEST="$HOME/.local/bin"
          mkdir -p "$DEST"
          export PATH="$DEST:$PATH"

          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -L -o "$DEST/docker-compose" \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x "$DEST/docker-compose"
          fi
          docker-compose version

          # known_hosts 登録（初回の Host key verification 対策）
          mkdir -p ~/.ssh
          if ! ssh-keygen -F 192.168.10.64 >/dev/null; then
            ssh-keyscan -H 192.168.10.64 >> ~/.ssh/known_hosts || true
          fi
        '''
      }
    }

    stage('Pre Check') {
      steps {
        script {
          def dockerConfigExists = sh(
            script: 'test -f ~/.docker/config.json && echo true || echo false',
            returnStdout: true
          ).trim()
          if (dockerConfigExists == 'true') {
            sh 'grep -H "docker.io" ~/.docker/config.json || true'
          } else {
            echo "Warning: Docker config not found. Skipping Docker login check."
          }
        }
      }
    }

    stage('Build') {
      steps {
        sh '''
          set -eux
          # docker-compose を見つけられるよう PATH を前置（各 sh ブロックで必要）
          export PATH="$HOME/.local/bin:$PATH"

          # Composeファイルを確認
          test -f docker-compose.build.yml && cat docker-compose.build.yml

          # DOCKER_HOST=ssh://... でリモートデーモンを指定
          export DOCKER_HOST=ssh://${BUILD_HOST}

          # 旧コンテナ/ボリューム整理（存在しなくてもOK）
          docker-compose -f docker-compose.build.yml down || true
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

