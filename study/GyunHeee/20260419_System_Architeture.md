# 2026. 04. 19. (일)

## Question

```
[배경]

- 드론 이착륙장(버티포트) CCTV 원격 관제 시스템 개발 중
- 서버 환경: 클라우드가 아닌 회사 온프레미스 로컬 서버 (단일 물리 서버)
- 배포 방식: Docker Compose 기반 컨테이너 운영
- 카메라: HIKVISION PTZ 카메라 3대, RTSP 프로토콜로 영상 송출

[현재 고려 중인 두 가지 구조]

구조 A - 단일 서버 (NestJS 통합 처리)
- NestJS 하나가 RTSP 수신, HLS 변환, 녹화, API 처리를 모두 담당

구조 B - 서버 분리
- Recorder(MediaMTX): RTSP 수신 및 24시간 녹화 전담
- NestJS: HLS 스트림 변환 및 브라우저 전달, 비즈니스 로직 처리 전담

[요구사항]

- 24시간 상시 녹화 필수
- 웹 브라우저를 통한 실시간 라이브 스트리밍
- PTZ 카메라 원격 제어 (ONVIF)
- NestJS 재배포 또는 장애 발생 시에도 녹화 연속성 보장

[질문]

1. 위 요구사항을 기준으로 두 구조 중 어떤 것이 더 적합한지
2. 분리 구조를 선택할 경우 발생할 수 있는 트레이드오프는 무엇인지
3. 온프레미스 단일 서버 환경에서 분리 구조 운영 시 주의해야 할 점은 무엇인지
```

## Answer

### 결론

**24시간 상시 녹화가 필수인 이상 구조 B(분리 구조)가 맞다.**
NestJS 재배포 시 녹화가 끊기는 문제는 단일 서버 구조에서 근본적으로 해결할 수 없기 때문이다.

---

### 구조 비교

| 구분 | 구조 A (단일) | 구조 B (분리) |
|---|---|---|
| RTSP 수신 | NestJS 내부 처리 | MediaMTX 전담 |
| HLS 변환 | NestJS FFmpeg 직접 실행 | MediaMTX 내부 파이프라인 |
| 장애 격리 | 단일 장애점 존재 | 서버 간 독립 |
| 재배포 영향 | 재배포 시 녹화 중단 | API 재배포 시 녹화 유지 |
| 자원 경쟁 | CPU/메모리 공유 | 영상 처리와 API 자원 독립 |
| 구성 복잡도 | 단순 (컨테이너 1개) | 중간 (컨테이너 2개 + 연동) |

---

### 구조 B 선택 시 트레이드오프

**얻는 것**
- NestJS 재배포·크래시 시 녹화 연속성 보장
- 영상 처리 자원과 API 자원 분리로 성능 안정
- MediaMTX on-connect webhook으로 NestJS 코드 변경 최소화

**잃는 것**
- Docker Compose 서비스 2개 관리 필요
- MediaMTX가 단일 장애점이 됨 → 별도 헬스체크 필수
- 초기 설정 복잡도 증가

---

### 온프레미스 단일 서버 환경에서 분리 구조 운영 시 주의사항

**1. MediaMTX 장애 대응**

MediaMTX가 죽으면 녹화와 라이브 스트림이 동시에 중단된다.
반드시 자동 재시작과 헬스체크를 설정해야 한다.

```yaml
# docker-compose.yml
recorder:
  image: bluenviron/mediamtx
  restart: always
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9997/v3/paths/list"]
    interval: 10s
    retries: 3
```

**2. 카메라 RTSP 세션 수 제한**

카메라 3대, MediaMTX + NestJS 각각 접속 시 카메라당 커넥션 2개 점유.
HIKVISION 보급형 기준 동시 세션 2~6개 제한이므로 현재 구조는 문제없으나,
운영자가 iVMS 등으로 직접 붙을 경우 3개로 증가 → 한계 도달 가능.

필요 시 MediaMTX re-publish 방식으로 커넥션을 1개로 줄일 수 있다.

```yaml
# mediamtx.yml
paths:
  cam1:
    source: rtsp://192.168.1.10:554/stream
    record: yes
    recordPath: /recordings/%path/%Y-%m-%d_%H-%M-%S
```

→ NestJS는 `rtsp://mediamtx:8554/cam1` 으로 접속
→ 카메라 커넥션 1개로 감소

**3. 스토리지 용량 관리**

24시간 상시 녹화 시 카메라 3대 기준 하루 수십~수백 GB 발생 가능.
현재 서버 HDD 1TB 기준 보존 기간 설정 필수.

```yaml
# mediamtx.yml
paths:
  cam1:
    recordDeleteAfter: 24h  # 24시간 이후 자동 삭제
```

**4. 로컬 서버 재부팅 대응**

서버 재부팅 시 Docker Compose가 자동으로 올라오도록 설정해야 한다.

```bash
# 서버 부팅 시 자동 실행 등록
sudo systemctl enable docker
# docker-compose.yml 내 모든 서비스에 restart: always 설정
```
