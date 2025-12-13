## Dockerfile
Dockerfile은 이미지를 생성하기 위한 설정을 정의하는 파일이다.  
컨테이너 실행 전 어떤 환경에서 어떤파일을 복사하고 어떤 명령을 실행할지 정의한다.

### FROM
베이스 이미지 생성
- 컨테이너를 생성할 때 컨테이너 안에 기본 프로그램으로 어떤 프로그램들이 깔려있어야 하는지 선택하는 옵션
ex) OS, 런타임 환경
- Docker Hub 사이트에서 이미지 확인 가능

```docker
# 도커 이미지(Docker Image)를 빌드할 때, JAVA 17 기반의 개발 및 실행 환경으로 도커 이미지를 빌드
FROM eclipse-temurin:17-jdk  
```

### COPY
파일 복사(이동)
- 호스트 -> 컨테이너로 파일/폴더를 복사
- ```.dockerignore``` 파일에 복사가 불필요한 파일을 지정하면 제외 가능

```docker
FROM ubuntu

# 현재 경로의 app.txt를 컨테이너의 절대경로 app.txt에 복사
COPY test.txt /test.txt 

# 폴더의 경우 뒤에 /추가  
COPY test-folder /test-folder/ 

# 와일드 카드로 모든 파일도 지정 가능
COPY *.txt /test-folder/ 

# 전체 복사
COPY ./ / 
```

### ENTRYPOINT
컨테이너가 시작될 때 실행되는 명령어
- 주로 서버 실행명령을 작성
- 컨테이너 = 하나의 프로세스 실행

```docker
FROM eclipse-temurin:17-jdk

COPY build/libs/*SNAPSHOT.jar app.jar

# 빌드된 spring 실행시키는 명령을 추가해서 컨테이너가 시작될때 스프링 서버를 시작시킴
ENTRYPOINT ["java", "-jar", "/app.jar"]  

# 도커 컨테이너를 실행시키고 localhost:8080접속시 잘 뜨는걸 확인할 수 있음.
docker run -d -p 8080:8080 hello-server
```


### RUN
이미지를 생성하는 과정에서 사용할 명령문 실행
- 이미지를 빌드할 때 한번 실행
- 패키지 설치, 빌드 작업에 사용
- 결과는 이미지 레이어로 저장됨

```docker
FROM ubuntu

# apt 패키지 관리자를 사용하여 시스템을 업데이트하고 'git'을 설치
RUN apt update && apt install -y git
```

### RUN vs ENTRYPOINT
|| RUN | ENTRYPOINT | note |
|----|----|----|----|
| 실행 시점 | 이미지 빌드 시 | 컨테이너 실행 시 |
| 목적 | 환경 준비 | 앱 실행 |
| 실행 횟수 | 1회 | 커넽이너 실행마다 |

### WORKDIR
작업 디렉토리 지정
- 컨테이너의 기존 파일들과 구분되게 디렉토리가 올라감
- 컨테이너 접속시 해당 디렉토리 경로로 접속됨
- 경로 관리를 깔끔하게 할 수 있음
- 디렉토리가 없으면 자동으로 생성
```docker
FROM ubuntu

# 도커 이미지 내부에서 앞으로 실행될 모든 명령어(예: RUN, CMD, COPY)의 기준 디렉터리를 /test-folder로 지정
WORKDIR /test-folder

# 도커파일이 있는 모든 소스 코드를 이미지의 /test-folder 안에 복사
COPY ./ ./
```

### EXPOSE
컨테이너 포트 문서화
- 컨테이너 내부에서 사용하는 포트 명시
- 실제 포트 개방이 아닌 문서화 목적
```Docker
# 도커 컨테이너 작동에는 영향을 미치지않음
# 이 이미지는 3000번 포트에서 도는걸 명시
EXPOSE 3000
```