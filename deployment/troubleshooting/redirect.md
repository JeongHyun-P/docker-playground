### 1. 링크를 클릭했을 때 실서버 URL이 아닌 localhost로 리다이렉트 되는 문제 발생

### 원인
- 도커 컨테이너 환경에서는 host 헤더가 정확하지 않다고 한다. next가 nginx에 의해 프록시 될때 auth.js가 origin을 잘못 추론.
- auth.js는 AUTH_URL 환경변수가 없으면 req.headers.host, req.headers.origin, fallback: localhost 등으로 origin을 결정한다
- next는 운영 환경에 배포할 때 사이트의 정식 URL을 환경변수로 설정해야한다고 한다.
https://next-auth.js.org/configuration/options
https://authjs.dev/reference/nextjs

### 해결
AUTH_URL 환경변수를 추가해서 해결.