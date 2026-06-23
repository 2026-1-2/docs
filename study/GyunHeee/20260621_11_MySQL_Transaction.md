# MySQL 트랜잭션 적용 시점

## 트랜잭션이 필요한 기준

여러 테이블에 걸친 작업 중 하나라도 실패하면 **전체를 롤백해야 일관성이 유지되는 경우**에 트랜잭션을 건다.
단일 테이블 INSERT/UPDATE는 MySQL이 자체적으로 원자성을 보장하므로 명시적 트랜잭션 불필요.

---

## 적용 시점 1: 카메라 등록 + MediaMTX path 등록

카메라를 DB에 저장한 후 MediaMTX 등록에 실패하면 DB에는 있지만 실제 스트림이 없는 유령 카메라가 생긴다.

```typescript
// camera.service.ts
async create(dto: CreateCameraDto): Promise<Camera> {
  return this.dataSource.transaction(async (manager) => {
    // 1. DB 저장
    const camera = manager.create(Camera, dto);
    await manager.save(camera);

    // 2. MediaMTX 등록 (실패 시 트랜잭션 전체 롤백)
    try {
      await this.mediaMtxService.registerPath(camera.name, camera.rtspUrl);
    } catch (error) {
      throw new Error(`MediaMTX 등록 실패: ${error.message}`);
      // 예외 throw → 트랜잭션 자동 롤백 → camera INSERT 취소
    }

    return camera;
  });
}
```

단, MediaMTX는 외부 시스템이므로 트랜잭션 롤백이 MediaMTX 상태를 되돌리지는 않는다.
DB 롤백 후 MediaMTX도 별도로 정리하는 보상 트랜잭션(compensating transaction)이 필요한 경우도 있다.

---

## 적용 시점 2: 녹화 시작 (recording + recording_segments 초기화)

```typescript
async startRecording(cameraId: number): Promise<Recording> {
  return this.dataSource.transaction(async (manager) => {
    // 1. recordings 레코드 생성
    const recording = manager.create(Recording, {
      cameraId,
      startedAt: new Date(),
      status: 'recording',
    });
    await manager.save(recording);

    // 2. 이벤트 로그 생성 (동시에 일어나야 함)
    const event = manager.create(RecordingEvent, {
      recordingId: recording.id,
      type: 'STARTED',
      occurredAt: new Date(),
    });
    await manager.save(event);

    return recording;
  });
}
```

---

## 적용 시점 3: 이벤트 감지 + 알람 생성

```typescript
async handleMotionDetected(cameraId: number, snapshotPath: string): Promise<void> {
  await this.dataSource.transaction(async (manager) => {
    // 1. events 테이블에 이벤트 저장
    const event = await manager.save(DetectionEvent, {
      cameraId,
      type: 'MOTION',
      snapshotPath,
      occurredAt: new Date(),
    });

    // 2. alerts 테이블에 알람 생성 (이벤트와 반드시 함께)
    await manager.save(Alert, {
      eventId: event.id,
      cameraId,
      status: 'UNREAD',
    });
  });
  // 트랜잭션 완료 후 SSE 푸시 (DB 저장 성공 확인 후)
  this.alertService.pushAlert({ cameraId, type: 'MOTION' });
}
```

이벤트는 저장됐는데 알람이 없거나, 알람은 있는데 이벤트가 없는 상황을 방지한다.

---

## TypeORM 트랜잭션 두 가지 방법

### 방법 1: dataSource.transaction (권장)
```typescript
await this.dataSource.transaction(async (manager) => {
  await manager.save(EntityA, dataA);
  await manager.save(EntityB, dataB);
});
```

### 방법 2: QueryRunner (세밀한 제어 필요 시)
```typescript
const queryRunner = this.dataSource.createQueryRunner();
await queryRunner.connect();
await queryRunner.startTransaction();

try {
  await queryRunner.manager.save(EntityA, dataA);
  await queryRunner.manager.save(EntityB, dataB);
  await queryRunner.commitTransaction();
} catch (error) {
  await queryRunner.rollbackTransaction();
  throw error;
} finally {
  await queryRunner.release();
}
```

---

## 트랜잭션을 걸지 않아도 되는 경우

- 단일 테이블 단일 행 INSERT/UPDATE
- 조회(SELECT)만 하는 작업
- 실패해도 재시도로 해결 가능한 독립적인 작업
