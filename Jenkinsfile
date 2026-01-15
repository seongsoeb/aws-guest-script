pipeline {
    agent any

    tools {
        gradle 'gradle-7.6.1'
        jdk 'jdk-17'
    }
    /***********************
     * 환경 설정
     ***********************/
    parameters {
        // DockerHub 사용자명 입력
        string(name: 'DOCKERHUB_USERNAME',  defaultValue: 'academyitwill', description: 'DockerHub 사용자명을 입력하세요.')
        // GitHub  사용자명 입력
        string(name: 'GITHUB_USERNAME',  defaultValue: '2025-07-JAVA-DEVELOPER-162', description: 'GitHub  사용자명을 입력하세요.')
    }
    environment {
       // ===== Git Repositories =====
        GITHUB_SCRIPT_URL = "https://github.com/${GITHUB_USERNAME}/aws-guest-script.git"
        GITHUB_SOURCE_URL = "https://github.com/${GITHUB_USERNAME}/aws-guest-source.git"
        
       // ===== EC2 =====
        EC2_USER='ubuntu'
        EC2_HOST='54.180.139.195'
        EC2_APP_DIR='/home/ubuntu/aws-guest-script'
        DOCKER_COMPOSE_FILE='docker-compose-guest-mysql-image.yml'
        //DOCKER_COMPOSE_FILE='docker-compose-guest-mysql.yml'
        // Jenkins → Credentials → SSH Username with private key
        EC2_SSH_CREDENTIALS   = 'deploy-ssh-key'
        // Jenkins → Credentials → Username with password (DockerHub 로그인용)
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
    }
    stages {
        /***********************
         * 1. 소스 체크아웃 & Gradle 빌드
         ***********************/
        stage('Source Build') {
            steps {
                git branch: 'main',
                url: "${GITHUB_SOURCE_URL}"
                sh "chmod +x ./gradlew"
                sh "./gradlew clean build"
            }
        }
        /***********************
         * 2.  Docker 이미지 빌드 & DockerHub 푸시
         ***********************/
        stage('Docker File Checkout Docker Build & Push') {
            steps {
               echo "==== Git Checkout 시작 ===="
               checkout([
                    $class: 'GitSCM',
                    branches: [[name: "*/main"]],
                    userRemoteConfigs: [[
                        url: "${GITHUB_SCRIPT_URL}"
                    ]]
               ])
                echo "==== Git Checkout 완료 ===="
                withCredentials([usernamePassword(
                    credentialsId: env.DOCKERHUB_CREDENTIALS,
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh """
                        docker build  -t ${params.DOCKERHUB_USERNAME}/guest .
                        echo "${DOCKERHUB_PASS}" | docker login -u "${DOCKERHUB_USER}" --password-stdin
                        docker push ${params.DOCKERHUB_USERNAME}/guest
                        docker logout
                    """
                    }
            }
        }

        /***********************
         * 1. EC2에 접속하여 git + docker-compose 배포
         ***********************/
        stage('EC2 배포') {
        steps {
        sshagent([env.EC2_SSH_CREDENTIALS]) {
            sh """
                ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                    set -e

                    echo "[EC2] 배포 디렉토리: ${EC2_APP_DIR}"
                    rm -rf "${EC2_APP_DIR}"
                    mkdir -p "${EC2_APP_DIR}"
                    cd "${EC2_APP_DIR}"

                    echo "[EC2] Git clone: ${params.GITHUB_USERNAME}/deploy_guest_aws_script.git"
                    git clone https://github.com/${params.GITHUB_USERNAME}/deploy_guest_aws_script.git .

                    echo "[EC2] 현재 디렉토리:"
                    pwd

                    echo "[EC2] 디렉토리 파일 목록:"
                    ls -al

                    echo "[EC2] docker-compose pull (이미지 갱신)"
                    docker compose -f ${DOCKER_COMPOSE_FILE} pull || true
                    
                    echo "[EC2] 기존 컨테이너 down"
                    docker compose -f ${DOCKER_COMPOSE_FILE} down || true

                   echo "[EC2] 기존 이미지 삭제 (compose에서 사용하는 이미지만)"
                    IMAGES=\$(docker compose -f ${DOCKER_COMPOSE_FILE} images -q || true)
                    if [ -n "\$IMAGES" ]; then
                        echo "[EC2] 삭제할 이미지: \$IMAGES"
                        docker rmi \$IMAGES || true
                    else
                        echo "[EC2] 삭제할 이미지 없음"
                    fi

                    echo "[EC2] Dangling 이미지 정리"
                    docker image prune -f || true

                    echo "[EC2] 새로운 컨테이너 up -d"
                    docker compose -f ${DOCKER_COMPOSE_FILE} up -d

                    echo "[EC2] 컨테이너 상태 확인"
                    docker compose -f ${DOCKER_COMPOSE_FILE} ps
                '
            """
        }
    }
}

    }
  
    post {
        success {
            echo "✅ EC2 배포 성공!"
        }
        failure {
            echo "❌ EC2 배포 실패. 콘솔 로그를 확인하세요."
        }
    }
}
