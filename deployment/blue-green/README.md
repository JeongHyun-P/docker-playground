## Deployment Blue-Green

-   AWS EC2 + ECR 기반 블루그린 배포
-   EC서버 ubuntu 기준

### 1. EC2 생성 및 Docker 설치

````bash
# timezone 설정
sudo timedatectl set-timezone Asia/Seoul
# NTP 활성화
sudo timedatectl set-ntp true

# 도커 설치
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
sudo usermod -aG docker ubuntu
exec $SHELL
````

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

-   AmazonEC2ContainerRegistryFullAccess 권한을 가진 IAM을 생성
-   AWS 외부 (로컬, EC2)에서 사용할 Access Key, Secret Key 발급
-   aws configure를 통해 로컬, EC2서버에 AWS CLI 인증 설정

### 4. ECR 리포지토리 생성

-   AWS 콘솔 → ECR → Repository 생성 ex)api

### 5. 디렉토리 구성 및 환경변수 작성

```bash
/home/ubuntu/api/
├── compose/
│   ├── api/
│   │   ├── blue.yml
│   │   └── green.yml
│   └── nginx/
│       └── nginx.yml
├── env/
│   ├── api_blue.env
│   └── api_green.env
├── nginx/
│   ├── nginx.conf
│   ├── api.conf
│   └── front.conf
├── certbot/
│   ├── conf
│   └── www
├── scripts/
│   └── deploy.sh
└── logs/
    ├── api_blue/
    └── api_green/

# 디렉토리 생성
cd /home/ubuntu
mkdir -p api
sudo chown -R ubuntu:ubuntu /home/ubuntu/api
sudo chmod -R 775 /home/ubuntu/api
cd /home/ubuntu/api
mkdir -p compose env nginx scripts logs

# 환경변수 작성
vi env/api_blue.env
vi env/api_green.env # 설정 같으면 그냥 cp env/api_blue.env env/api_green.env
```

### 7. Docker 네트워크 생성

도커 네트워크 없이 컨테이너를 실행하면 컨테이너는 랜덤 IP를 받고 재시작할때마다 IP가 바뀌어 nginx설정을 매번 수정해야한다.
도커 네트워크를 쓰면 컨테이너가 같은 가상 네트워크에 묶이고 IP가 바뀌어도 이름은 그대로

```bash
docker network create api_net  # 네트워크 생성
docker network ls  # 확인
```

### 8. docker-compose.yml 작성

nginx.yml

```yml
services:
    nginx:
        image: nginx:1.25-alpine
        container_name: nginx
        ports:
            - "80:80"
            - "443:443"
        volumes:
            - ../../nginx/nginx.conf:/etc/nginx/nginx.conf:ro
            - ../../nginx/api.conf:/etc/nginx/conf.d/api.conf:ro
        networks:
            - api_net
        restart: always

networks:
    api_net:
        external: true
```

blue.yml

```yml
services:
    api_blue:
        image: 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com/api
        container_name: api_blue
        env_file:
            - ../../env/api_blue.env
        volumes:
            - ../../logs/api_blue:/api/logs
            - /etc/localtime:/etc/localtime:ro
            - /etc/timezone:/etc/timezone:ro
        networks:
            - api_net
        restart: always

networks:
    api_net:
        external: true
```

green.yml

```yml
services:
    api_green:
        image: 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com/api
        container_name: api_green
        env_file:
            - ../../env/api_green.env
        volumes:
            - ../../logs/api_green:/api/logs
            - /etc/localtime:/etc/localtime:ro
            - /etc/timezone:/etc/timezone:ro
        networks:
            - api_net
        restart: always

networks:
    api_net:
        external: true
```

### 9. nginx 설정파일 작성

nignx/nginx.conf

```nginx
user  nginx;
worker_processes auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format main
      '$remote_addr - $remote_user [$time_local] "$request" '
      '$status $body_bytes_sent "$http_referer" '
      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile on;
    tcp_nopush on;
    keepalive_timeout 65;

    server_tokens off;

    include /etc/nginx/conf.d/*.conf;
}
```

nignx/api.conf

```nginx
upstream api_upstream {
    server api_blue:4001;
    # server api_green:4001;
}

server {
    listen 80;
    client_max_body_size 50M;
    client_body_timeout 5s;
    server_name api.dockerbg.r-e.kr;

    location / {
        return 503 "api not ready";
    }
}

```

### 10. Dockerfile 작성 및 이미지 빌드 및 푸시

Dockerfile (nest)

```docker
FROM node:22-alpine AS builder
WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install --frozen-lockfile

COPY . .

RUN yarn build

FROM node:22-alpine AS runner

WORKDIR /app

COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json yarn.lock ./

CMD ["node", "dist/main.js"]
```

-   AWS콘솔에서 생성한 ECR의 푸시 명령보기를 참고하여 푸시명령을 진행하면됨.

1. Docker 클라이언트 인증
   `aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com`

2. 도커 이미지 빌드
   `docker build -t api .`
   !!! 빌드할때 EC2와 아키텍쳐가 다르면 오류가 발생함. 로컬PC가 Mac Apple Silicon일 경우 플랫폼을 지정해서 빌드해야함 !!!
   `docker build --platform linux/amd64 -t api .`

3. 빌드 완료 후 이미지에 태그 지정
   `docker tag api:latest 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com/api:latest`

4. ECR에 이미지 푸시
   `docker push 11111111111.dkr.ecr.ap-northeast-2.amazonaws.com/api:latest`

### 10. 컨테이너 실행

```bash
cd /home/ubuntu/api

# blue 최초 배포
docker compose -f compose/api/blue.yml up -d

# nginx
docker compose -f compose/nginx/nginx.yml up -d

# nginx 상태 확인
docker exec nginx nginx -t
```

### 11. 인증서 발급

1. comopse/nginx/nginx.yml 설정에 webroot 마운트 추가

```yml
volumes:
    - ../../certbot/www:/var/www/certbot
    - ../../certbot/conf:/etc/letsencrypt
```

2. nginx 재가동
   `docker compose -f compose/nginx/nginx.yml up -d`

3. nginx/api.conf에 인증용 location 임시로 추가

```nginx

...

server {
    listen 80;
    server_name api.dockerbg.r-e.kr;

    # =========추가=========================
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
        allow all;
    }
    # =====================================

    location / {
        proxy_pass http://api_upstream;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

4. nginx 컨테이너 다시 올린 후 /var/www/certbot확인
   `docker compose -f compose/nginx/nginx.yml down`
   `docker compose -f compose/nginx/nginx.yml up -d`
   `docker exec nginx ls -l /var/www/certbot`

5. certbot 실행

```bash
docker run --rm \
  -v /home/ubuntu/api/certbot/conf:/etc/letsencrypt \
  -v /home/ubuntu/api/certbot/www:/var/www/certbot \
  certbot/certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email {email} \
  --agree-tos \
  --no-eff-email \
  -d api.dockerbg.r-e.kr
```

6. nginx/api.conf에 HTTPS 서버블록 추가 후 재시작

```nginx
upstream api_upstream {
    server api_blue:4001;
    # server api_green:4001;
}

# HTTPS
server {
    listen 443 ssl;
    server_name api.dockerbg.r-e.kr;

    ssl_certificate /etc/letsencrypt/live/api.dockerbg.r-e.kr/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.dockerbg.r-e.kr/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    location / {
        proxy_pass http://api_upstream;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }

    location /nginx-health {
        return 200 'ok';
        add_header Content-Type text/plain;
    }
}

# HTTP → HTTPS redirect
server {
    listen 80;
    server_name api.dockerbg.r-e.kr;
    return 301 https://$host$request_uri;
}
```
docker compose -f compose/nginx/nginx.yml down
docker compose -f compose/nginx/nginx.yml up -d

### 12. blue <-> green 전환

nginx/api.conf의 upstream blue<->green 전환하고 docker exec nginx nginx -s reload

### 13. 배포 자동화를 위한 스크립트 작성
scripts/deploy.sh
```sh
#!/usr/bin/env bash
# ==========================================
# Auto Blue-Green Deploy Script (latest only)
# ==========================================
# - 현재 nginx upstream 상태를 읽어서
#   ▶ 비활성 색상으로 배포
#   ▶ 헬스체크 통과 시 자동 스위치
# - Nginx 컨테이너는 reload만 수행 → 무중단

set -e

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PROJECT_NAME="api"
API_DIR="/home/ubuntu/api"
NGINX_CONF="$API_DIR/nginx/api.conf"
NGINX_CONTAINER="nginx"
BACK_PORT="4001"
HEALTH_PATH="/health-check"
LOCK_FILE="$SCRIPT_DIR/deploy.lock"

# ------------------------------------------
# Lock (중복 실행 방지)
# ------------------------------------------
if [ -f "$LOCK_FILE" ]; then
  echo "❌ deploy already running"
  exit 1
fi
trap "rm -f $LOCK_FILE" EXIT
touch "$LOCK_FILE"

# ------------------------------------------
# 현재 활성 색상 판단
# ------------------------------------------
if grep -Eq "^[[:space:]]*server api_blue" "$NGINX_CONF"; then
  CURRENT="blue"
  NEXT="green"
elif grep -Eq "^[[:space:]]*server api_green" "$NGINX_CONF"; then
  CURRENT="green"
  NEXT="blue"
else
  echo "❌ cannot determine current color"
  exit 1
fi

echo "current color: $CURRENT"
echo "next color: $NEXT"

# ------------------------------------------
# 다음 색상 컨테이너 배포 (latest)
# ------------------------------------------
COMPOSE_FILE="$API_DIR/compose/api/$NEXT.yml"

echo "▶ $NEXT Container Deploy (latest)"
docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" pull
docker compose -p "$PROJECT_NAME" -f "$COMPOSE_FILE" up -d

# ------------------------------------------
# 헬스 체크 대기
# ------------------------------------------
NEXT_CONTAINER="api_$NEXT"

echo "▶ $NEXT waiting health check"
for i in {1..20}; do
  if docker exec "$NEXT_CONTAINER" node -e "
    require('http')
      .get('http://localhost:$BACK_PORT$HEALTH_PATH', res => {
        process.exit(res.statusCode === 200 ? 0 : 1);
      })
      .on('error', () => process.exit(1));
  "; then
    echo "✅ completed health check"
    break
  fi

  echo "... waiting ($i)"
  sleep 3

  if [ "$i" -eq 20 ]; then
    echo "❌ failed health check"
    exit 1
  fi
done

# ------------------------------------------
# nginx upstream 자동 전환
# ------------------------------------------
echo "▶ nginx upstream change: $CURRENT → $NEXT"

# 임시 파일 생성 후 원본에 덮어쓰기
TEMP_CONF=$(mktemp)

sed -E "s/^[[:space:]]*server api_$CURRENT/# server api_$CURRENT/" "$NGINX_CONF" | \
sed -E "s/^[[:space:]]*#\s*server api_$NEXT/server api_$NEXT/" > "$TEMP_CONF"

# inode를 유지하면서 내용만 변경
cat "$TEMP_CONF" > "$NGINX_CONF"
rm -f "$TEMP_CONF"

docker exec "$NGINX_CONTAINER" nginx -t && \
docker exec "$NGINX_CONTAINER" nginx -s reload

# ------------------------------------------
# 이전 컨테이너 Down
# ------------------------------------------
echo "▶ stop old container: api_$CURRENT"
docker compose -p "$PROJECT_NAME" -f "$API_DIR/compose/api/$CURRENT.yml" down

echo "✅ Deploy Complete: Current Color → $NEXT"
```

스크립트 실행 확인
chmod +x deploy.sh
./deploy.sh

---

## CI/CD with GitHub Actions
1. GitHub Actions 등록 스크립트 실행 완료 후 runner 서비스 등록 및 자동시작 설정
```bash
# 설치 및 서비스 등록
./svc.sh install
./svc.sh start
```

2. 워크플로우 파일 작성
.github/workflows/deploy-prod.yml
```yml
# 러너 잘 도는지 테스트
name: deploy-prod

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    runs-on: [self-hosted, Linux, X64, deploy-prod]

    steps:
    - name: Checkout repository
      uses: actions/checkout@v3

    - name: Print Working Directory
      run: |
        echo "Runner is working!"
        pwd
        whoami
        hostname
```

러너 확인 후 내용 수정
```yml
name: deploy-prod

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest   # GitHub Actions Hosted runner에서 빌드

    steps:
      # 레포지토리 코드 체크아웃
      - name: Checkout repository
        uses: actions/checkout@v3

      # Yarn 캐시 활용
      - name: Cache Yarn dependencies
        uses: actions/cache@v3
        with:
          path: |
            ~/.cache/yarn
          key: yarn-${{ hashFiles('**/yarn.lock') }}
          restore-keys: yarn-

      # AWS 자격 증명 구성
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ secrets.AWS_REGION }}

      # Amazon ECR에 로그인
      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1

      # Docker 이미지 빌드 및 태그
      - name: Build Docker image
        run: |
          docker build --platform linux/amd64 -t api:latest .
          docker tag api:latest ${{ secrets.ECR_REPOSITORY_API }}:latest

      # Docker 이미지를 ECR에 푸시
      - name: Push Docker image to ECR
        run: |
          docker push ${{ secrets.ECR_REPOSITORY_API }}:latest

  deploy-on-ec2:
    needs: build-and-push
    runs-on: self-hosted # 자체 호스팅된 EC2 인스턴스에서 배포
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      # EC2 인스턴스의 배포 스크립트 실행
      - name: Pull and run latest Docker image
        run: |
          cd ~/api
          aws ecr get-login-password --region ap-northeast-2 | docker login --username AWS --password-stdin  ${{ secrets.ECR_REPOSITORY_API }}
          docker pull ${{ secrets.ECR_REPOSITORY_API }}:latest
          chmod +x scripts/deploy.sh
          ./scripts/deploy.sh
```

