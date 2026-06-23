# 시스템 장애 발생 시 관리자 알림 흐름

## 알림이 필요한 장애 유형

| 장애 유형 | 트리거 | 심각도 |
|----------|--------|--------|
| 카메라 스트림 끊김 | MediaMTX webhook | warning |
| 카메라 장기 오프라인 (10분+) | NestJS 폴링 | critical |
| 디스크 사용량 90% 초과 | NestJS cron | warning |
| 디스크 사용량 95% 초과 | NestJS cron | critical |
| is_flyable 상태 변경 | 환경 모니터링 | critical |
| mmp-server 헬스체크 실패 | Docker healthcheck | critical |

---

## 알림 전달 경로

```
장애 이벤트 발생
    ↓
NestJS AlertService.pushAlert()
    ↓
    ├── alerts 테이블 INSERT (DB 기록)
    │
    └── Subject.next() (RxJS)
            ↓
    SSE 연결 중인 모든 React 클라이언트에 즉시 전달
            ↓
    관리자 화면 알람 패널에 실시간 표시
```

---

## AlertService 구현

```typescript
// alert/alert.service.ts
import { Injectable } from '@nestjs/common';
import { Subject } from 'rxjs';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { Alert } from './alert.entity';

@Injectable()
export class AlertService {
  private readonly alertStream$ = new Subject<Alert>();

  constructor(
    @InjectRepository(Alert)
    private readonly alertRepo: Repository<Alert>,
  ) {}

  async pushAlert(data: {
    type: string;
    cameraId?: number;
    vertiportId?: number;
    message: string;
    severity: 'info' | 'warning' | 'critical';
  }): Promise<void> {
    // 1. DB에 저장
    const alert = await this.alertRepo.save({
      ...data,
      status: 'unread',
      createdAt: new Date(),
    });

    // 2. SSE로 실시간 전달
    this.alertStream$.next(alert);
  }

  getAlertStream() {
    return this.alertStream$.asObservable();
  }
}
```

---

## 각 장애별 알림 트리거 코드

### 카메라 끊김 (webhook)

```typescript
// webhook.controller.ts
@Post('stream-down')
async handleStreamDown(@Body() body: { name: string }) {
  const camera = await this.cameraService.findByName(body.name);
  await this.alertService.pushAlert({
    type: 'STREAM_DOWN',
    cameraId: camera.id,
    message: `카메라 [${camera.name}] 스트림 연결이 끊겼습니다`,
    severity: 'warning',
  });
}
```

### 디스크 부족 (cron)

```typescript
// storage/storage-monitor.service.ts
@Cron('0 * * * *')  // 매시간
async checkDiskUsage(): Promise<void> {
  const usagePercent = await this.getDiskUsagePercent('/mnt/hdd1');

  if (usagePercent >= 95) {
    await this.alertService.pushAlert({
      type: 'DISK_CRITICAL',
      message: `HDD 사용량 위험: ${usagePercent}% (즉시 조치 필요)`,
      severity: 'critical',
    });
  } else if (usagePercent >= 90) {
    await this.alertService.pushAlert({
      type: 'DISK_WARNING',
      message: `HDD 사용량 경고: ${usagePercent}%`,
      severity: 'warning',
    });
  }
}
```

### 장기 오프라인 감지 (폴링)

```typescript
// camera-health.service.ts
@Interval(60 * 1000)
async checkLongOfflineCameras(): Promise<void> {
  const tenMinutesAgo = new Date(Date.now() - 10 * 60 * 1000);

  const longOffline = await this.cameraRepo.find({
    where: {
      status: 'offline',
      lastHealthCheck: LessThan(tenMinutesAgo),
    },
  });

  for (const camera of longOffline) {
    await this.alertService.pushAlert({
      type: 'CAMERA_LONG_OFFLINE',
      cameraId: camera.id,
      message: `카메라 [${camera.name}] 10분 이상 응답 없음`,
      severity: 'critical',
    });
  }
}
```

---

## React에서 SSE 알람 수신 및 표시

```typescript
// hooks/useAlerts.ts
useEffect(() => {
  const eventSource = new EventSource('/api/alerts/stream', {
    withCredentials: true,
  });

  eventSource.onmessage = (e) => {
    const alert = JSON.parse(e.data);

    // critical은 팝업 + 소리
    if (alert.severity === 'critical') {
      showModal(alert);
      playAlertSound();
    }

    // 알람 패널에 추가
    setAlerts((prev) => [alert, ...prev]);
  };

  return () => eventSource.close();
}, []);
```

---

## 외부 알림 연동 (선택적 확장)

SSE는 브라우저가 열려있을 때만 받을 수 있다.
브라우저를 닫은 상태에서도 알림을 받으려면 외부 채널이 필요하다.

```typescript
// 심각도 critical 알림을 이메일/슬랙으로도 발송
if (data.severity === 'critical') {
  await this.emailService.send({
    to: 'admin@example.com',
    subject: `[긴급] ${data.type}`,
    body: data.message,
  });
}
```

프로젝트 규모에 따라 Slack webhook, 이메일, SMS 등을 추가할 수 있다.
