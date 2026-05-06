# 2026. 05. 05. (화)

## Question

```
[현재 구조]

- Recorder(MediaMTX): 카메라 RTSP에 직접 접속 → 24시간 녹화 담당
- NestJS: 카메라 RTSP에 별도 접속 → HLS 변환 후 브라우저에 라이브 스트림 전달
- 결과적으로 카메라 1대당 RTSP 커넥션이 2개 존재

[분리한 이유]

- 장애 격리 목적: Recorder 다운 시 라이브 스트림 유지, NestJS 재시작 시 녹화 유지

[질문]

1. 위 구조에서 실제로 문제가 발생할 수 있는 시나리오가 있는지
2. 있다면 어떤 상황에서, 어떤 형태로 문제가 나타나는지
3. 장애 격리라는 목적은 유지하면서 커넥션 구조를 개선할 수 있는 아키텍처 방향이 있는지
```

## Answer

### 결론

**현재 2개 커넥션 구조 자체는 문제없다.** 오히려 장애 격리라는 목적에 부합하는 좋은 설계다.
유일한 실질적 위험은 **카메라의 RTSP 동시 세션 수 제한**이라는 하드웨어 제약 하나뿐이다.

---

### 장애 격리 시나리오 분석

| 장애 상황 | Recorder | NestJS(라이브) |
|---|---|---|
| Recorder 크래시 | 녹화 중단 | **라이브 스트림 유지 ✓** |
| NestJS 재시작/재배포 | **녹화 계속 유지 ✓** | 라이브 일시 끊김 |
| 카메라 세션 초과 | 둘 다 영향 받을 수 있음 | 둘 다 영향 받을 수 있음 |

→ 의도한 대로 **한쪽이 죽어도 다른 쪽은 살아있는 구조**가 정상 동작한다.

---

### 유일한 위험: 카메라 RTSP 세션 수 제한

HIKVISION 보급형 카메라의 동시 RTSP 세션 수는 모델에 따라 **2~6개**로 제한된다.

현재 사용 중인 카메라 기준:

- `DS-2DE2C400MWG-E` (PTZ): 보통 3~6개
- `DS-2SE3C204MWG-E/12`: 보통 2~4개

현재 구조에서 카메라 1대당 커넥션:

- Recorder 1개 + NestJS 1개 = **2개 상시 점유**
- 여기에 운영자가 현장에서 iVMS 앱이나 웹뷰어로 직접 붙으면 → **3개로 한계에 도달** 가능

---

### 개선 방향 (필요한 경우에만)

세션 수가 빠듯하다면 **MediaMTX re-publish 방식**으로 전환한다.
카메라 커넥션은 MediaMTX 1개만 유지하고, NestJS는 카메라 대신 MediaMTX 내부 RTSP를 구독한다.

```
Camera → MediaMTX(Recorder) → NestJS (내부 RTSP 구독)
                            → Client (HLS 직접 재생)
```

`mediamtx.yml` 설정 예시:

```yaml
paths:
  cam1:
    source: rtsp://192.168.1.10:554/stream
    record: yes
    recordPath: /recordings/%path/%Y-%m-%d_%H-%M-%S
```

- NestJS: `rtsp://mediamtx:8554/cam1` 으로 접속
- 클라이언트: `http://mediamtx:8888/cam1/index.m3u8` 으로 HLS 재생
- 카메라 커넥션: **1개로 감소**

단, 이 경우 **MediaMTX가 단일 장애점**이 되므로 반드시 아래 설정이 필요하다:

```yaml
# docker-compose.yml
recorder:
  restart: always
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:9997/v3/paths/list"]
    interval: 10s
    retries: 3
```

---

### 정리

- 세션 수가 **충분하다면** → 현재 구조 유지, 바꿀 필요 없음
- 세션 수가 **빠듯하다면** → MediaMTX re-publish 방식으로 전환
- 판단 기준: 카메라 웹 UI → 네트워크 설정에서 최대 동시 스트림 수 확인
