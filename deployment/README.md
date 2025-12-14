## Deployment
- AWS EC2 + ECR 기반 배포
- EC서버 ubuntu 기준

### ECR
- AWS 환경에서 Docker 이미지를 안전하게 저장 및 배포하기 위한 AWS 네이티브 컨테이너 레지스트리, 보안/배포 자동화/운영 안정성이 뛰어남
- 서버에서 코드를 내려받고 빌드하는 방식이 아니라 서버 상태나 라이브러리 버전에 따라 실행결과가 다른문제 해결
- 빌드를 서버에서 하지 않기때문에 빌드할때 발생하는 CPU,메모리 장애 유발 해결
- 롤백문제 해결 코드를 내려받고 빋르하면 배포된 상태에서 코드를 덮어씌우거나 돌려서 이전 빌드 결과는 사라지는데 장애시 이전상태로 되돌릴 수 있음.
- CI또는 로컬에서 이미지 빌드 -> ECR에 빌드한 이미지 푸시 -> EC2는 이미지만 ECR에서 pull해서 실행 -> 서버는 실행만 담당

### 1. EC2 생성 및 Docker 설치
```bash
sudo apt-get update && \
sudo apt-get install -y ca-certificates curl gnupg lsb-release && \
sudo install -m 0755 -d /etc/apt/keyrings && \
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null && \
sudo apt-get update && \
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 설치 확인
docker -v
docker compose version

# docker 권한 설정
```sudo usermod -aG docker ubuntu```
```exec $SHELL```
```

### 2. AWS CLI 설치
```bash
sudo apt install unzip
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# 설치 확인
aws --version
```

### 3. AWS IAM 설정
- AmazonEC2ContainerRegistryFullAccess 권한을 가진 IAM을 생성
- AWS 외부 (로컬, EC2)에서 사용할 Access Key, Secret Key 발급
- aws configure를 통해 로컬, EC2서버에 AWS CLI 인증 설정


### 4. ECR 리포지토리 생성
- AWS 콘솔 → ECR → Repository 생성 ex)front, back

### 5. Dockerfile 작성 및 이미지 빌드 및 푸시
- AWS콘솔에서 생성한 ECR의 푸시 명령보기를 참고하여 푸시명령을 진행하면됨.
1. Docker 클라이언트 인증
```aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com```

2. 도커 이미지 빌드
```docker build -t front .```
!!! 빌드할때 EC2와 아키텍쳐가 다르면 오류가 발생함. 로컬PC가 Mac Apple Silicon일 경우 플랫폼을 지정해서 빌드해야함 !!!
```docker build --platform linux/amd64 -t front .```

4. 빌드 완료 후 이미지에 태그 지정
```docker tag front:latest 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com/front:latest```

4. ECR에 이미지 푸시
```docker push 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com/front:latest```

### 6. EC2에서 ECR 이미지 Pull
1. Docker 클라이언트 인증
```aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com```

2. 이미지 Pull
```docker compose pull```

### 7. compose.yml or docker-compose.yml 작성
```yml
services:
  front:
    image: 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com/front
    container_name: front # 컨테이너 이름 지정
    restart: always  # 컨테이너가 죽으면 자동으로 재시작
    env_file:
      - ./.env.front  # 환경변수는 EC서버에서 파일로 만들어 둔 뒤 통째로 컨테이너에 주입
    expose:
      - "3000" # 포트 정보 명시 Docker 네트워크 내부에서 3000 오픈 (다른 컨테이너에서 접근 가능)
    ports:
      - "3000:3000"  # 포트 매핑
    healthcheck:  # 컨테이너 내부에서 헬스 체크 진행 
      test: ["CMD-SHELL", "wget -qO- http://0.0.0.0:3000/api/health || exit 1"]  # wget -qO- http://0.0.0.0:3020/api/health 명령어 실행
      interval: 5s  # 5초 주기
      retries: 5  # 5번 실패시 unhealthy 판정
      start_period: 30s # 해당 시간만큼 컨테이너 시작 후 헬스체크 실패해도 무시
```

### 8. 실행
```docker compose up -d```