pipeline {
    agent any
    environment {
        DOCKERHUB_USER = "fujiwarakousuke"
        BUILD_HOST = "root@192.168.10.64"
        PROD_HOST  = "root@192.168.10.64"
    }
    stages {
        stage('Pre Check') {
            steps {
                script {
                    // Docker config ファイルが存在するか確認
                    def dockerConfigExists = sh(
                        script: 'test -f ~/.docker/config.json && echo true || echo false',
                        returnStdout: true
                    ).trim()
                    
                    if (dockerConfigExists == 'true') {
                        sh 'cat ~/.docker/config.json | grep docker.io || true'
                    } else {
                        echo "Warning: Docker config not found. Skipping Docker login check."
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    def timestamp = sh(script: "date +%Y%m%d-%H%M%S", returnStdout: true).trim()
                    
                    sh "cat docker-compose.build.yml"
                    sh "docker compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml down || true"
                    sh "docker -H ssh://${BUILD_HOST} volume prune -f || true"
                    sh "docker compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml build"
                    sh "docker compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml up -d"
                    sh "docker compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml ps"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_apptest pytest -v test_app.py"
                    sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_webtest pytest -v test_static.py"
                    sh "docker -H ssh://${BUILD_HOST} container exec dockerkvs_webtest pytest -v test_selenium.py"
                    sh "docker compose -H ssh://${BUILD_HOST} -f docker-compose.build.yml down || true"
                }
            }
        }

        stage('Register') {
            steps {
                script {
                    def timestamp = sh(script: "date +%Y%m%d-%H%M%S", returnStdout: true).trim()
                    sh "docker -H ssh://${BUILD_HOST} tag dockerkvs_web ${DOCKERHUB_USER}/dockerkvs_web:${timestamp}"
                    sh "docker -H ssh://${BUILD_HOST} push ${DOCKERHUB_USER}/dockerkvs_web:${timestamp}"
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh "ssh ${PROD_HOST} 'docker pull ${DOCKERHUB_USER}/dockerkvs_web:${timestamp} && docker compose -f docker-compose.prod.yml up -d'"
                }
            }
        }
    }
}

