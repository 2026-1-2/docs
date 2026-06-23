# RTSP 연결 끊김/복구 상황 재현 테스트

## 실제 카메라 없이 RTSP 스트림 모의하는 방법

### FFmpeg로 가짜 RTSP 소스 만들기

```bash
# 테스트용 영상 파일을 RTSP 스트림으로 송출
ffmpeg -re -stream_loop -1 \
  -i test_video.mp4 \
  -c copy \
  -f rtsp \
  rtsp://localhost:8554/test-camera

# 또는 컬러 바 영상(영상 파일 없이)으로 송출
ffmpeg -re \
  -f lavfi -i testsrc=size=1280x720:rate=30 \
  -f lavfi -i sine=frequency=440 \
  -c:v libx264 -c:a aac \
  -f rtsp rtsp://localhost:8554/test-camera
```

MediaMTX가 이 주소를 소스로 바라보게 하면, 실제 카메라 없이 스트림이 생성된다.

```yaml
# mediamtx.yml
paths:
  test-camera:
    source: rtsp://ffmpeg-source:8554/test-camera
```

---

## 연결 끊김 재현

### 방법 1: FFmpeg 프로세스 강제 종료

```bash
# 터미널 1: FFmpeg로 스트림 송출
ffmpeg -re -stream_loop -1 -i test.mp4 -f rtsp rtsp://localhost:8554/test-camera &
FFMPEG_PID=$!

# 터미널 2: 테스트 실행 중 30초 후 강제 종료 (끊김 재현)
sleep 30 && kill -9 $FFMPEG_PID

# 60초 후 재시작 (복구 재현)
sleep 60 && ffmpeg -re -stream_loop -1 -i test.mp4 -f rtsp rtsp://localhost:8554/test-camera
```

### 방법 2: Docker 네트워크 차단으로 끊김 재현

```bash
# MediaMTX와 카메라 소스 간 네트워크 일시 차단
docker network disconnect mmp-network ffmpeg-source

# 복구
sleep 30 && docker network connect mmp-network ffmpeg-source
```

---

## 재연결 자동화 검증

MediaMTX가 자동으로 재연결을 시도하는지 확인한다.

```bash
# MediaMTX 로그에서 재연결 시도 확인
docker compose logs -f mediamtx | grep -E "reconnect|error|ready"
```

기대되는 로그 흐름:
```
[cam1] source not ready
[cam1] retrying in 5s...
[cam1] retrying in 10s...
[cam1] source is ready  ← 재연결 성공
```

---

## webhook 이벤트 발생 확인

끊김/복구 시 NestJS webhook이 제대로 수신되는지 확인한다.

```bash
# NestJS 로그에서 webhook 수신 확인
docker compose logs -f mmp-server | grep webhook

# 또는 webhook 엔드포인트에 임시 로깅 추가
# POST /webhook/stream-down 수신 시 console.log
```

---

## 자동화 테스트 스크립트

```bash
#!/bin/bash
# rtsp-reconnect-test.sh

echo "=== RTSP 재연결 테스트 시작 ==="

# 1. FFmpeg 스트림 시작
docker run -d --name test-stream --network mmp-network \
  linuxserver/ffmpeg \
  -re -stream_loop -1 -i /test.mp4 \
  -f rtsp rtsp://mediamtx:8554/test-camera

sleep 5

# 2. 스트림 정상 여부 확인
STATUS=$(curl -s http://localhost:9997/v3/paths/get/test-camera | jq '.ready')
echo "초기 상태: $STATUS"  # true여야 함

# 3. 끊김 재현
echo "스트림 중단..."
docker stop test-stream

sleep 5

STATUS=$(curl -s http://localhost:9997/v3/paths/get/test-camera | jq '.ready')
echo "끊김 후 상태: $STATUS"  # false여야 함

# 4. 복구 재현
echo "스트림 재시작..."
docker start test-stream

sleep 15  # 재연결 대기

STATUS=$(curl -s http://localhost:9997/v3/paths/get/test-camera | jq '.ready')
echo "복구 후 상태: $STATUS"  # true여야 함

# 정리
docker rm -f test-stream
echo "=== 테스트 완료 ==="
```

---

## 검증 포인트 체크리스트

- [ ] 스트림 끊김 후 MediaMTX가 자동 재연결 시도하는가
- [ ] 끊김 시 NestJS에 `stream-down` webhook이 오는가
- [ ] 복구 시 NestJS에 `stream-ready` webhook이 오는가
- [ ] React 화면에서 연결 상태가 실시간으로 바뀌는가
- [ ] 끊김 동안 DB에 알람 레코드가 생성되는가
