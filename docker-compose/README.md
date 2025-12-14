## Docker Compose
여러 컨테이너를 하나의 서비스로 정의하고 구성  
```docker run``` 명령을 YAML 파일로 선언적으로 관리.
- 여러 컨테이너를 한 번에 실행 및 종료
- 서비스간 의존성 관리

### 주요 속성
| 속성 | 기능 | note | 
|----|----|----|
| services | 실행할 컨테이너 정의 | 서비스명은 컨테이너간 통신시 호스트이름으로 사용된다. |
| image | 사용할 이미지 지정 |  |
| build | Dockerfile 기반 이미지 빌드 |  |
| ports | 호스트-컨테이너 포트 매핑 |  |
| environment | 환경 변수 설정 |  |
| volumes | 호스트-컨테이너간 디렉토리 공유 | 컨테이너 삭제 후에도 데이터가 유지됨 |
| depends_on | 의존성 설정 |  |
| healthcheck | 컨테이너 상태 확인 |  |

```yaml
# 하나의 컨테이너를 services로 정의
# nginx
services:
  nginx-server:
    container_name: nginx-server
    image: nginx 
    ports:
      - 80:80
      
# mysql      
services:
  my-db:
    image: mysql
    # 환경변수
    environment:
      MYSQL_ROOT_PASSWORD: pwd1234
    volumes:
      - ./mysql_data:/var/lib/mysql
    ports:do
      - 3307:3306
      
# spring
services:
  my-server:
    build: . # 이미지를 도커파일을 기반으로한 이미지로 빌드 (도커파일이 있는 디렉토리를 값으로 지정)
    ports:
      - 8080:8080
```


--- 
### Docker Compose 명령어
| 명령어 | 설명 | note |
| ---- | --- | --- |
| ```docker compose up``` | ```docker-compose.yml``` 또는 ```compose.yml``` 에서 정의한 파일을 기반으로 컨테이너를 실행 |  |
| ```docker compose up -d``` | 컨테이너를 백그라운드로 실행 |  |
| ```docker compose down``` | 컨테이너 실행 종료 |  |
| ```docker compose ps``` | compose단위 컨테이너들의 상태 확인 |  |
| ```docker compose logs``` | compose단위 컨테이너들의 로그 확인 |  |
| ```docker compose up --build``` | 이미지를 빌드해서 실행 |  |
| ```docker compose pull``` | 도커허브에 있는 이미지를 다운받거나 버전 업데이트 |  |
| ```docker compose up -d --build``` | ```docker-compose.yml``` 또는 ```compose.yml``` 에서 정의한 빌드 실행 |  |
