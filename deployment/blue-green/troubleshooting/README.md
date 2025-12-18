### 1. Docker 볼륨 마운트 파일 동기화 이슈
Blue-Green 배포 시 호스트의 `nginx/api.conf` 파일은 정상적으로 수정되지만 nginx 컨테이너 내부의 파일은 변경되지 않아 upstream 전환이 실패하는 문제

### 원인
Docker 볼륨 마운트는 **파일 내용 변경**은 실시간으로 반영하지만 `sed -i` 명령어는 다음과 같이 동작하여 문제가 발생
1. 새로운 임시 파일 생성
2. 변경된 내용을 임시 파일에 작성
3. 원본 파일 삭제
4. 임시 파일을 원본 파일명으로 이동

이 과정에서 **inode가 변경**되어 컨테이너 내부의 마운트 포인트가 원본 파일과의 연결을 잃어버리게된다.

### 해결
`sed -i` 대신 **inode를 유지**하면서 파일 내용만 변경하는 방식 사용:
```bash
# ❌ 잘못된 방법 (inode 변경)
sed -i 's/pattern/replacement/' /path/to/file

# ✅ 올바른 방법 (inode 유지)
TEMP_CONF=$(mktemp)
sed 's/pattern/replacement/' /path/to/file > "$TEMP_CONF"
cat "$TEMP_CONF" > /path/to/file
rm -f "$TEMP_CONF"
```

- Docker 볼륨은 파일 시스템 레벨에서 inode를 추적
- `sed -i`는 atomic write를 위해 새 파일을 생성
- `cat > file` 방식은 기존 파일의 inode를 유지하며 내용만 덮어씀
