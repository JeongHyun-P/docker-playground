# Troubleshooting
AWS EC2 + ECR 기반 Docker 배포 과정에서 발생한 문제와 해결 방법 정리

- EC2 디렉토리 구성
```
/app
  ├ docker-compose.yml
  ├ .env.front
  ├ .env.back
```

- Dockerfile
```docker
# next.js
FROM node:22-alpine AS builder

WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install --frozen-lockfile

COPY . .

ARG NEXT_PUBLIC_CLOUDFRONT_URL=""
ENV NEXT_PUBLIC_CLOUDFRONT_URL=$NEXT_PUBLIC_CLOUDFRONT_URL

RUN yarn build

FROM node:22-alpine AS runner

WORKDIR /app

COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3000

CMD ["node", "server.js"]

# ==============================================================

# nest.js
FROM node:22-alpine AS builder
WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install --frozen-lockfile

COPY . .

RUN yarn build

FROM node:22-alpine AS runner

COPY package.json yarn.lock ./
RUN yarn install --frozen-lockfile --production

COPY --from=builder /app/dist ./dist

EXPOSE 3001
CMD ["node", "dist/main.js"]
```

- docker-comopse.yml
```yml
services:

  front:
    image: 111111111111.dkr.ecr.ap-northeast-2.amazonaws.com/front
    container_name: front
    restart: always
    env_file:
      - ./.env.front
    expose:
      - "3000"
    ports:
      - "3000:3000"
    healthcheck:
      test: ["CMD-SHELL", "wget -qO- http://0.0.0.0:3000/api/health || exit 1"]
      interval: 5s
      retries: 5
      start_period: 30s

  back:
    image: 111111111111.dkr.ecr.ap-northeast-2.amazonaws.com/back
    container_name: back
    restart: always
    env_file:
      - ./.env.back
    expose:
      - "3001"
    ports:
      - "3001:3001"
    healthcheck:
      test: ["CMD-SHELL", "nc -z localhost 3001"]
      interval: 5s
      retries: 5
      start_period: 30s
    volumes:
      - ./logs:/app/logs

  nginx:
    image: nginx:1.25
    container_name: nginx_proxy
    restart: always
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./certbot/conf:/etc/letsencrypt:ro
    depends_on:
      front:
        condition: service_healthy
      back:
        condition: service_healthy

  certbot:
    image: certbot/certbot
    container_name: certbot
    volumes:
      - ./certbot/www:/var/www/certbot
      - ./certbot/conf:/etc/letsencrypt
    entrypoint: >
      sh -c "trap exit TERM; while :; do sleep 12h & wait $${!}; certbot renew --webroot -w /var/www/certbot; done"
```