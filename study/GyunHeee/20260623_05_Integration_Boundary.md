# NestJS ↔ MediaMTX 연동 필요한 부분 vs 불필요한 부분

## 핵심 기준

> **NestJS가 "명령"을 내려야 하는 것만 연동한다. MediaMTX가 스스로 판단하는 것은 건드리지 않는다.**

---

## 연동 필요한 부분 — NestJS → MediaMTX API 호출

| 기능 | NestJS 행동 | MediaMTX API |
|------|------------|-------------|
| 카메라 등록 | RTSP URL + 이름 전달 | `POST /v3/config/paths/add/{name}` |
| 카메라 수정 | 변경된 URL 재전달 | `PATCH /v3/config/paths/patch/{name}` |
| 카메라 삭제 | path 제거 요청 | `DELETE /v3/config/paths/delete/{name}` |
| 헬스체크 | 스트림 상태 조회 | `GET /v3/paths/list` |
| 스냅샷 요청 | 현재 프레임 요청 | (별도 처리, 후술) |

이것들은 **사용자 액션이 트리거**하는 동작이다. NestJS가 판단해서 MediaMTX에게 지시해야 한다.

---

## 연동 불필요한 부분 — MediaMTX가 알아서 처리

| 기능 | 이유 |
|------|------|
| 실제 RTSP 연결 | MediaMTX가 자체적으로 카메라에 접속 |
| HLS/WebRTC 변환 | 내장 트랜스코딩 엔진이 처리 |
| 재연결 로직 | MediaMTX 설정(`runOnDisconnect` 등)으로 자동 처리 |
| .ts 세그먼트 파일 생성/삭제 | MediaMTX가 파일시스템 직접 관리 |
| 클라이언트 세션 관리 | MediaMTX 내부 처리 |

NestJS가 이 영역에 개입하면 **책임 충돌**이 발생한다.

---

## 경계를 헷갈리게 만드는 것들

### "헬스체크는 NestJS가 폴링해야 하나?"
아니다. NestJS는 필요할 때 `GET /v3/paths/list`를 조회한다.
MediaMTX는 연결이 끊기면 webhook으로 NestJS에게 알려준다 (→ topic 8 참고).

### "스냅샷은 MediaMTX가 만드나?"
MediaMTX 자체 스냅샷 기능은 제한적이다.
일반적으로 FFmpeg 또는 OpenCV로 별도 처리하고, 결과 파일을 chokidar가 감지한다 (→ topic 9 참고).

---

## 한 줄 기억법

```
NestJS = 카메라 "등록대장" 관리자
MediaMTX = 영상 "실제 처리" 담당자
```

등록대장에 이름 올리고 지우는 건 NestJS 몫.
실제 영상 처리는 MediaMTX가 알아서 한다.
