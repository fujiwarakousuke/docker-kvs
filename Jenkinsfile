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
          # docker-compose 単体バイナリを用意（無ければ取得）
          if ! command -v docker-compose >/dev/null 2>&1; then
            curl -L -o /usr/local/bin/docker-compose \
              "https://github.com/docker/compose/releases/download/v2.27.0/docker-compose-linux-x86_64"
            chmod +x /usr/local/bin/docker-compose
          fi
          docker-compose version

          # known_hosts 登録（初回の Host key verification failed 対策）
          mkdir -p ~/.ssh
          if ! ssh-keygen -F 192.168.10.64 >/dev/null; then
            ssh-keyscan -H 192.168.10.64 >> ~/.ssh/known_hosts
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
          # Composeファイルはワークスペース直下にある前提（無ければ失敗するので先に cat で確認）
          cat docker-compose.build.yml

          # DOCKER_HOST=ssh://... でリモートデーモンを指定
          export DOCKER_HOST=ssh://${BUILD_HOST}

          # 旧コンテナ/ボリューム整理（あってもなくてもOK）
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

