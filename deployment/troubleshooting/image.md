### 1. tandard_init_linux.go:228: exec user process caused: exec format error

### 원인
로컬에서 빌드한 이미지는 Mac ARM, EC2는 x86_64 운영체제가 서로 달라서 오류가 남.

### 해결
로컬에서 빌드시 플랫폼을 지정해서 빌드
```docker build --platform linux/amd64 -t front .```