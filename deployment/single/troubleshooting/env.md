### 1. Next에서 설정한 일부 환경변수가 서버에 있는 환경변수로 바뀌지않고 브라우저에서 로컬에서 빌드할때의 환경변수로 실행이됨. 
서버에서 환경변수를 출력해보면 환경변수는 서버에서 지정한 값으로 잘 출력됨.
- 컨테이너 내부 접속
```docker exec -it {Container ID} sh ```
- 환경변수 출력
``` echo $NEXT_PUBLIC_CLOUDFRONT_URL ```

### 원인
NEXT의 NEXT_PUBLIC_* 환경변수는 빌드시점에 주입해서 고정해 빌드타임에 결정되는 정적 환경변수이다.
빌드시점에 컴파일되어 포함되므로 런타임에 변경되지 않는다. 이후에는 불변이므로 컨테이너 런타임에서 변경되어도 반영되지 않은것

- next 공식문서 참고
https://nextjs.org/docs/14/app/building-your-application/configuring/environment-variables



### 해결
Dockerfile에 빌드 아규먼트(ARG)를 지정해 이미지 빌드시점에 값을 주입하도록 함

```zsh
docker build \
--platform linux/amd64 \
--build-arg NEXT_PUBLIC_CLOUDFRONT_URL=https://xxxxxxxxxxx.cloudfront.net \
-t front .
```

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package.json yarn.lock ./

RUN yarn install --frozen-lockfile

COPY . .

# ========================추가==============================
ARG NEXT_PUBLIC_CLOUDFRONT_URL=""  
ENV NEXT_PUBLIC_CLOUDFRONT_URL=$NEXT_PUBLIC_CLOUDFRONT_URL
# =========================================================

RUN yarn build

FROM node:22-alpine AS runner

WORKDIR /app

COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public

EXPOSE 3020

CMD ["node", "server.js"]
```
