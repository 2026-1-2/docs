# 다수 카메라 동시 스트리밍 시 서버 부하 측정

## 측정 목표

"카메라 몇 대까지 동시에 스트리밍 가능한가"와
"어느 시점에 어떤 리소스(CPU/메모리/네트워크)가 병목인가"를 파악한다.

---

## 가짜 카메라 N개 동시 생성

```bash
#!/bin/bash
# spawn-cameras.sh
# 인자로 카메라 수를 받음: bash spawn-cameras.sh 20

N=${1:-10}

for i in $(seq 1 $N); do
  docker run -d \
    --name test-cam-$i \
    --network mmp-network \
    linuxserver/ffmpeg \
    -re -stream_loop -1 \
    -f lavfi -i testsrc=size=1280x720:rate=15 \
    -c:v libx264 -preset ultrafast -b:v 500k \
    -f rtsp rtsp://mediamtx:8554/cam$i

  # MediaMTX에 path 등록
  curl -s -X POST "http://localhost:9997/v3/config/paths/add/cam$i" \
    -H "Content-Type: application/json" \
    -d "{\"source\": \"rtsp://test-cam-$i:8554/cam$i\"}"

  echo "카메라 $i 기동"
done

echo "$N개 카메라 모두 기동 완료"
```

---

## 서버 리소스 모니터링

### 실시간 컨테이너 리소스 확인

```bash
# 모든 컨테이너 CPU/메모리 실시간 확인
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
```

```
NAME            CPU %     MEM USAGE       NET I/O
mediamtx        45.2%     312MiB/8GiB    1.2GB/800MB
mmp-server      8.1%      180MiB/8GiB    50MB/20MB
mysql           3.4%      256MiB/8GiB    10MB/5MB
```

### 시간별 리소스 기록

```bash
#!/bin/bash
# monitor.sh - 30초마다 스냅샷 기록
while true; do
  echo "=== $(date) ===" >> resource_log.txt
  docker stats --no-stream --format \
    "{{.Name}},{{.CPUPerc}},{{.MemPerc}},{{.NetIO}}" >> resource_log.txt
  sleep 30
done
```

---

## 부하 단계별 측정 방법

```
카메라 5대 → 10분 모니터링 → 리소스 기록
카메라 10대 → 10분 모니터링 → 리소스 기록
카메라 20대 → 10분 모니터링 → 리소스 기록
카메라 30대 → 한계 도달 여부 확인
```

각 단계에서 측정하는 지표:

```bash
# CPU 사용률
docker stats --no-stream mediamtx | awk 'NR==2 {print $3}'

# 네트워크 처리량 (MediaMTX가 핵심)
# 카메라 1대당 500kbps → 20대 = 10Mbps 수신 + N명 시청자 × 500kbps 송신

# 메모리 사용량
docker stats --no-stream mediamtx | awk 'NR==2 {print $4}'
```

---

## MediaMTX 처리 한계 측정

MediaMTX가 병목이 되는 시점을 확인한다.

```bash
# 스트림 상태 전체 조회 (active 스트림 수 확인)
curl -s http://localhost:9997/v3/paths/list | jq '.items | length'

# 각 스트림의 reader 수 (동시 시청자)
curl -s http://localhost:9997/v3/paths/list | jq '.items[].readers'
```

---

## 동시 시청자 부하 테스트

카메라 수뿐 아니라 "동시 시청자" 수도 테스트한다.

```bash
# k6로 HLS 스트림 동시 접속 테스트
# https://k6.io

# k6 스크립트 (load-test.js)
import http from 'k6/http';
import { sleep } from 'k6';

export const options = {
  vus: 50,         // 가상 사용자 50명 동시
  duration: '2m',
};

export default function () {
  // HLS 플레이리스트 주기적 요청 (실제 HLS 플레이어 동작 모사)
  http.get('http://localhost:8888/cam1/index.m3u8');
  sleep(2);  // 2초마다 세그먼트 요청
  http.get('http://localhost:8888/cam1/segment_latest.ts');
}
```

```bash
k6 run load-test.js
```

---

## 측정 결과 정리 예시

| 카메라 수 | MediaMTX CPU | 메모리 | 네트워크 in | 관찰 |
|----------|-------------|--------|------------|------|
| 5대 | 8% | 120MB | 2.5Mbps | 안정 |
| 10대 | 18% | 200MB | 5Mbps | 안정 |
| 20대 | 42% | 380MB | 10Mbps | 안정 |
| 30대 | 78% | 620MB | 15Mbps | 간헐적 지연 |
| 40대 | 95%+ | 850MB | - | 스트림 끊김 발생 |

---

## 병목 지점에 따른 대응

| 병목 | 증상 | 대응 |
|------|------|------|
| MediaMTX CPU | CPU 100%, 스트림 지연 | 컨테이너 CPU 코어 수 증가, 해상도/프레임 낮춤 |
| 네트워크 대역폭 | 패킷 손실, 버퍼링 | NIC 업그레이드, 해상도 낮춤 |
| 메모리 | OOMKilled | 메모리 증설, 동시 스트림 수 제한 |
| NestJS | API 응답 지연 | 수평 확장, 쿼리 최적화 |
