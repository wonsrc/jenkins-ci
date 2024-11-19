// 필요한 변수를 선언할 수 있다. (내가 직접 선언하는 변수, 젠킨스 환경변수를 끌고 올 수 있음)
def dockerHubUsername = "wonsrc"
def dockerHubRepository="my-app-image"  // 도커 이미지 이름
def deployHost="172.31.16.30"  // 배포서버 private IP

// 젠킨스의 선언형 파이프라인 정의부 시작 (그루비 언어)
pipeline {
    agent any // 어느 젠킨스 서버에서도 실행이 가능

    // docker hub에 로그인을 스테이지 별로 두 번 진행.
    stages {
        stage('Pull Codes from Github'){ // 스테이지 제목 (맘대로 써도 됨.)
            steps{
                checkout scm // 젠킨스와 연결된 소스 컨트롤 매니저(git 등)에서 코드를 가져오는 명령어
            }
        }
        stage('Build Codes by Gradle') {
            steps {
                sh """
                ./gradlew clean build
                ls -al ./app/build/libs
                """
            }
        }
        stage('Retrieve Credentials') {
            steps {
                withCredentials([string(credentialsId: 'DOCKER_HUB_PASSWORD', variable: 'DOCKER_PASSWORD')]) {
                    sh """
                    # Docker Hub 비밀번호를 환경 변수로 설정
                    echo "${DOCKER_PASSWORD}" > .docker_password
                    """
                }
            }
        }
        stage('Build Docker Image & Push to Docker Hub') {
            steps {
                sh """
                    # Docker Hub 로그인
                    docker login -u "${dockerHubUsername}" -p \$(cat .docker_password)

                    # Docker 이미지 빌드
                    docker build -t ${dockerHubUsername}/${dockerHubRepository}:${currentBuild.number} .

                    # Docker Hub로 푸시
                    docker push ${dockerHubUsername}/${dockerHubRepository}:${currentBuild.number}

                    # 민감한 정보 삭제
                    rm -f .docker_password

                """
            }
        }
        stage('Deploy to AWS EC2 VM') {
            steps {
                sshagent(credentials: ["jenkins-ssh-key"]) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${deployHost} \
                     'docker login -u ${dockerHubUsername} -p \$(cat .docker_password); \
                      docker pull ${dockerHubUsername}/${dockerHubRepository}:${currentBuild.number}; \
                      docker stop \$(docker ps -q) || true; \
                      docker run -d -p 80:8080 ${dockerHubUsername}/${dockerHubRepository}:${currentBuild.number};'

                    # 민감한 정보 삭제
                    rm -f .docker_password

                    """
                }
            }
        }
    }
}