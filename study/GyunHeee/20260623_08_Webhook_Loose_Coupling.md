# 트리거 방식의 연동 (느슨한 결합)

## 방향이 두 가지다

```
NestJS ──────────→ MediaMTX   (API 호출, pull 방식)
MediaMTX ─────────→ NestJS   (webhook, push 방식)
```

한쪽은 NestJS가 주도하고, 반대쪽은 MediaMTX가 주도한다.
이 두 방향을 구분하는 것이 핵심이다.

---

## NestJS → MediaMTX: API 호출

NestJS가 먼저 행동해야 할 때 사용한다.

```
카메라 등록 요청 → POST /v3/config/paths/add/{name}
카메라 삭제 요청 → DELETE /v3/config/paths/delete/{name}
스트림 상태 확인 → GET /v3/paths/get/{name}
```

사용자가 버튼을 누르거나, 스케줄이 실행되거나 — NestJS가 트리거가 되는 상황이다.

---

## MediaMTX → NestJS: Webhook (push)

MediaMTX에서 어떤 **이벤트가 발생**했을 때 NestJS에게 알려주는 방식이다.

### 폴링(polling)과의 차이

| | 폴링 | Webhook |
|--|------|---------|
| 주도권 | NestJS가 계속 물어봄 | MediaMTX가 발생 시점에 알려줌 |
| 네트워크 | 불필요한 요청 발생 | 이벤트 발생 시에만 요청 |
| 실시간성 | 폴링 주기만큼 지연 | 즉시 |

### MediaMTX webhook 이벤트 예시

```yaml
# mediamtx.yml
paths:
  cam1:
    source: rtsp://...
    runOnReady: curl -X POST http://nestjs:3000/webhook/stream-ready -d '{"name":"cam1"}'
    runOnNotReady: curl -X POST http://nestjs:3000/webhook/stream-down -d '{"name":"cam1"}'
    runOnRead: curl -X POST http://nestjs:3000/webhook/viewer-connected -d '{"name":"cam1"}'
```

스트림이 준비되면, 끊기면, 시청자가 접속하면 — 각각 NestJS의 특정 엔드포인트를 호출한다.

### NestJS에서 받는 쪽

```typescript
@Post('webhook/stream-down')
async handleStreamDown(@Body() body: { name: string }) {
  await this.alertService.createAlert(body.name, 'STREAM_DOWN');
}
```

NestJS는 webhook을 받아서 알람을 생성하거나 DB에 기록하는 등 비즈니스 로직을 처리한다.

---

## 느슨한 결합인 이유

- MediaMTX는 NestJS의 내부 구조를 모른다. HTTP POST를 보낼 URL만 알면 된다.
- NestJS는 MediaMTX가 어떻게 스트림을 관리하는지 모른다. webhook을 받으면 된다.
- 둘 중 하나가 잠깐 다운돼도 나머지는 계속 동작한다.

이게 **느슨한 결합(loose coupling)**이다.
