# last_health_check는 누가 어떤 주기로 갱신하는가

## 두 가지 접근 방식 비교

| | NestJS 폴링 | MediaMTX webhook |
|--|------------|-----------------|
| 방식 | NestJS가 주기적으로 MediaMTX API 조회 | MediaMTX가 상태 변경 시 NestJS에 push |
| 실시간성 | 폴링 주기만큼 지연 | 즉시 |
| 네트워크 비용 | 불필요한 요청 발생 | 이벤트 발생 시에만 |
| 카메라 수 증가 시 | 폴링 비용 선형 증가 | 영향 없음 |

---

## 실제 구현: 두 방식 조합

끊김 감지는 webhook(즉시), 정기 상태 확인은 폴링(주기적)으로 역할을 나눈다.

### 역할 1: MediaMTX webhook으로 상태 변경 즉시 감지

```yaml
# mediamtx.yml
paths:
  cam1:
    runOnReady: >
      curl -s -X POST http://mmp-server:3000/webhook/stream-ready
      -H "X-Webhook-Secret: ${WEBHOOK_SECRET}"
      -H "Content-Type: application/json"
      -d '{"name":"cam1"}'
    runOnNotReady: >
      curl -s -X POST http://mmp-server:3000/webhook/stream-down
      -H "X-Webhook-Secret: ${WEBHOOK_SECRET}"
      -H "Content-Type: application/json"
      -d '{"name":"cam1"}'
```

```typescript
// webhook.controller.ts
@Post('stream-ready')
async handleStreamReady(@Body() body: { name: string }) {
  await this.cameraService.updateHealthCheck(body.name, 'online');
}

@Post('stream-down')
async handleStreamDown(@Body() body: { name: string }) {
  await this.cameraService.updateHealthCheck(body.name, 'offline');
}
```

```typescript
// camera.service.ts
async updateHealthCheck(cameraName: string, status: 'online' | 'offline') {
  await this.cameraRepo.update(
    { name: cameraName },
    {
      status,
      lastHealthCheck: new Date(),
    },
  );
}
```

스트림이 끊기거나 복구되는 순간 `last_health_check`와 `status`가 즉시 갱신된다.

---

### 역할 2: NestJS 폴링으로 webhook 누락 보완

webhook이 전달되지 않는 경우(네트워크 일시 장애 등)를 대비한 보완 메커니즘이다.

```typescript
// camera-health.service.ts
@Injectable()
export class CameraHealthService {

  @Interval(60 * 1000)  // 1분마다
  async syncHealthStatus(): Promise<void> {
    // MediaMTX에서 현재 활성 스트림 목록 조회
    const { data } = await firstValueFrom(
      this.httpService.get('/v3/paths/list'),
    );

    const activePaths = new Set(
      data.items
        .filter((item: any) => item.ready)
        .map((item: any) => item.name),
    );

    const cameras = await this.cameraRepo.find();

    for (const camera of cameras) {
      const isOnline = activePaths.has(camera.name);
      const currentStatus = isOnline ? 'online' : 'offline';

      // 상태가 달라진 경우에만 업데이트 (불필요한 쓰기 방지)
      if (camera.status !== currentStatus) {
        await this.cameraRepo.update(camera.id, {
          status: currentStatus,
          lastHealthCheck: new Date(),
        });
      } else {
        // 상태 변화 없어도 last_health_check 시각은 갱신
        await this.cameraRepo.update(camera.id, {
          lastHealthCheck: new Date(),
        });
      }
    }
  }
}
```

---

## last_health_check 컬럼 설계

```sql
ALTER TABLE cameras ADD COLUMN last_health_check DATETIME;
ALTER TABLE cameras ADD COLUMN status ENUM('online', 'offline', 'unknown') DEFAULT 'unknown';
```

`last_health_check`가 오래됐다는 것 자체가 이상 신호다.

```typescript
// 10분 이상 갱신되지 않은 카메라 감지
const staleThreshold = new Date(Date.now() - 10 * 60 * 1000);
const staleCameras = await this.cameraRepo.find({
  where: { lastHealthCheck: LessThan(staleThreshold) },
});
```

---

## 흐름 요약

```
정상 상태:
  MediaMTX → runOnReady webhook → NestJS → last_health_check 갱신 (즉시)

1분마다:
  NestJS → GET /v3/paths/list → 전체 카메라 상태 동기화 (보완)

webhook 누락 시:
  1분 후 폴링에서 상태 불일치 감지 → 자동 보정
```
