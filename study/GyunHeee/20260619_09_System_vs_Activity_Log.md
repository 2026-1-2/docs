# system_log vs activity_log 분리 이유와 실제 활용

## 두 테이블이 답하는 질문이 다르다

| | system_log | activity_log |
|--|-----------|-------------|
| 질문 | "시스템에 무슨 일이 일어났는가?" | "사용자가 무엇을 했는가?" |
| 주체 | 시스템, 자동화 프로세스 | 사람(관리자, 운용자) |
| 예시 | 카메라 끊김, 디스크 경고, 재연결 | 카메라 등록, 설정 변경, 로그인 |
| 활용 목적 | 장애 원인 추적, 인프라 모니터링 | 감사(audit), 책임 추적 |

---

## 테이블 구조

```sql
-- 시스템 이벤트 로그
CREATE TABLE system_logs (
  id          INT PRIMARY KEY AUTO_INCREMENT,
  level       ENUM('info', 'warning', 'error', 'critical') NOT NULL,
  source      VARCHAR(100) NOT NULL,   -- 'MediaMTX', 'CameraHealth', 'StorageMonitor'
  event_type  VARCHAR(100) NOT NULL,   -- 'STREAM_DOWN', 'DISK_WARNING'
  camera_id   INT,
  message     TEXT NOT NULL,
  metadata    JSON,                    -- 추가 컨텍스트 (풍속값, 디스크 사용량 등)
  occurred_at DATETIME NOT NULL
);

-- 사용자 행동 로그
CREATE TABLE activity_logs (
  id          INT PRIMARY KEY AUTO_INCREMENT,
  user_id     INT NOT NULL,
  action      VARCHAR(100) NOT NULL,   -- 'CREATE_CAMERA', 'DELETE_RECORDING', 'LOGIN'
  target_type VARCHAR(50),             -- 'camera', 'recording', 'user'
  target_id   INT,
  ip_address  VARCHAR(45),
  user_agent  VARCHAR(255),
  request_body JSON,                   -- 변경 전/후 값
  occurred_at DATETIME NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(id)
);
```

---

## NestJS에서 로그 기록

### system_log: 자동으로 기록

```typescript
// system-log.service.ts
@Injectable()
export class SystemLogService {
  async log(data: {
    level: string;
    source: string;
    eventType: string;
    cameraId?: number;
    message: string;
    metadata?: object;
  }) {
    await this.systemLogRepo.save({
      ...data,
      occurredAt: new Date(),
    });
  }
}

// camera-health.service.ts에서 사용
await this.systemLogService.log({
  level: 'warning',
  source: 'CameraHealth',
  eventType: 'STREAM_DOWN',
  cameraId: camera.id,
  message: `카메라 [${camera.name}] 스트림 끊김`,
  metadata: { lastSeenAt: camera.lastHealthCheck },
});
```

### activity_log: 인터셉터로 자동 기록

```typescript
// interceptors/activity-log.interceptor.ts
@Injectable()
export class ActivityLogInterceptor implements NestInterceptor {
  constructor(private readonly activityLogService: ActivityLogService) {}

  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const request = context.switchToHttp().getRequest();
    const { method, url, body, ip, user } = request;

    // 쓰기 작업(POST/PATCH/DELETE)만 기록
    if (['POST', 'PATCH', 'DELETE'].includes(method)) {
      this.activityLogService.log({
        userId: user?.id,
        action: this.inferAction(method, url),
        ipAddress: ip,
        requestBody: body,
        occurredAt: new Date(),
      });
    }

    return next.handle();
  }

  private inferAction(method: string, url: string): string {
    if (url.includes('/cameras') && method === 'POST') return 'CREATE_CAMERA';
    if (url.includes('/cameras') && method === 'DELETE') return 'DELETE_CAMERA';
    // ...
    return `${method} ${url}`;
  }
}
```

---

## 실제 문제 추적 사례

### 사례 1: system_log로 장애 원인 파악

```
증상: 새벽 2시에 cam3 스트림이 끊겼다가 5분 만에 복구됨

system_log 조회:
  01:58  DISK_WARNING   HDD 사용량 89%
  02:00  STREAM_DOWN    cam3 끊김
  02:05  STREAM_READY   cam3 복구

→ 디스크 부족으로 세그먼트 파일 쓰기 실패 → 스트림 일시 중단
→ cron 삭제 작업 실행 후 자동 복구

원인: retention_days 정책이 너무 길었음 → 30일 → 14일로 변경
```

```sql
-- 특정 시간대 system_log 조회
SELECT * FROM system_logs
WHERE occurred_at BETWEEN '2026-06-19 01:50:00' AND '2026-06-19 02:10:00'
ORDER BY occurred_at;
```

### 사례 2: activity_log로 책임 추적

```
증상: cam5가 설정도 없이 삭제됨, 담당자가 모른다고 함

activity_log 조회:
  2026-06-18 16:32  user_id=7  DELETE_CAMERA  camera_id=5  ip=192.168.1.55

→ user_id=7 (홍길동 계정)으로 삭제 확인
→ 해당 사용자에게 확인 후 실수 판명
→ 감사 추적(audit trail)으로 분쟁 해결
```

```sql
-- 특정 카메라에 대한 모든 행동 조회
SELECT al.*, u.name
FROM activity_logs al
JOIN users u ON al.user_id = u.id
WHERE al.target_type = 'camera' AND al.target_id = 5
ORDER BY al.occurred_at DESC;
```

---

## 분리의 실질적 효과

하나의 `logs` 테이블에 섞으면:
- 초당 수십 건의 시스템 이벤트가 사용자 행동 로그를 묻어버림
- 인덱스 설계가 두 용도를 동시에 만족시킬 수 없음
- 보관 기간이 다름 (system_log 30일 vs activity_log 1년)

분리하면 각 테이블의 인덱스, 보관 정책, 조회 패턴을 독립적으로 최적화할 수 있다.
