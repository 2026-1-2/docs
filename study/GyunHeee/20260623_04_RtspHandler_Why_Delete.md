# 기존 RtspHandler 코드가 왜 통째로 삭제 대상인가

## 핵심 요약

기존 `RtspHandler` 클래스는 **MediaMTX가 이미 제공하는 기능을 NestJS에서 중복 구현한 것**이다.
MediaMTX를 도입하는 순간 이 클래스는 존재 이유가 사라진다.

---

## 기존 코드가 하던 일

```
RtspHandler
├── child_process.spawn('ffmpeg', [...])   // RTSP → HLS 변환
├── MAX_RETRIES = 3                        // 재시도 로직
├── 재연결 대기 타이머                      // 연결 끊김 감지 + 재시도
└── 세그먼트 파일 관리                     // .ts 파일 생성/삭제 추적
```

직접 FFmpeg 프로세스를 띄우고, 실패하면 다시 띄우고, 생성되는 `.ts` 세그먼트 파일을 직접 관리하는 구조다.

---

## 왜 삭제해야 하는가

### MediaMTX가 이 모든 걸 대신한다

| 기능 | 기존 RtspHandler | MediaMTX |
|------|-----------------|----------|
| RTSP → HLS/WebRTC 변환 | FFmpeg 직접 실행 | 내장 처리 |
| 연결 끊김 재시도 | MAX_RETRIES + 타이머 | 자동 재연결 |
| 세그먼트 파일 관리 | 직접 추적/삭제 | 자동 관리 |
| 다중 프로토콜 지원 | 불가 | HLS/WebRTC/RTMP 동시 지원 |

### 중복 구현의 문제

1. **버그 가능성 2배** — NestJS 재시도 로직과 MediaMTX 재연결이 동시에 동작하면 충돌
2. **리소스 낭비** — NestJS 프로세스가 FFmpeg를 직접 관리하면 Node.js 이벤트 루프 블로킹 위험
3. **유지보수 부담** — FFmpeg 옵션, 세그먼트 설정이 코드 안에 하드코딩되어 변경이 어려움

### 삭제 후 NestJS의 역할

```
삭제 전: NestJS → FFmpeg 직접 실행 → HLS 파일 생성
삭제 후: NestJS → MediaMTX API 호출 → MediaMTX가 FFmpeg 실행 및 파일 관리
```

NestJS는 스트리밍 로직에서 완전히 손을 뗀다.
카메라 URL 등록/삭제를 MediaMTX에 알려주는 것으로 역할이 끝난다.

---

## 결론

`RtspHandler`는 MediaMTX 없던 시절의 임시 구현체다.
코드 품질의 문제가 아니라, **아키텍처가 바뀌면서 계층 자체가 사라진 것**이다.
리팩토링이 아니라 **삭제**가 맞는 이유다.
