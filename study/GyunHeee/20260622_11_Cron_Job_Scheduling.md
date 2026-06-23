# NestJS에서 Cron Job / 스케줄링 구현

## 패키지 설치

```bash
npm install @nestjs/schedule
npm install -D @types/cron
```

---

## 모듈 등록

```typescript
// app.module.ts
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [
    ScheduleModule.forRoot(),  // 한 번만 등록
  ],
})
export class AppModule {}
```

---

## 오래된 녹화 파일 삭제 예시

`storage_policies` 테이블에 보관 기간(retention_days)이 정의되어 있다고 가정.

```typescript
// recording/recording-cleanup.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { InjectRepository } from '@nestjs/typeorm';
import { LessThan, Repository } from 'typeorm';
import { Recording } from './recording.entity';
import { StoragePolicy } from '../storage/storage-policy.entity';
import { promises as fs } from 'fs';

@Injectable()
export class RecordingCleanupService {
  private readonly logger = new Logger(RecordingCleanupService.name);

  constructor(
    @InjectRepository(Recording)
    private readonly recordingRepo: Repository<Recording>,
    @InjectRepository(StoragePolicy)
    private readonly policyRepo: Repository<StoragePolicy>,
  ) {}

  // 매일 새벽 3시 실행
  @Cron(CronExpression.EVERY_DAY_AT_3AM)
  async deleteExpiredRecordings() {
    this.logger.log('오래된 녹화 파일 삭제 시작');

    const policies = await this.policyRepo.find();

    for (const policy of policies) {
      const expiryDate = new Date();
      expiryDate.setDate(expiryDate.getDate() - policy.retentionDays);

      const expired = await this.recordingRepo.find({
        where: {
          camera_id: policy.cameraId,
          startedAt: LessThan(expiryDate),
        },
      });

      for (const recording of expired) {
        try {
          await fs.unlink(recording.filePath);
          await this.recordingRepo.remove(recording);
          this.logger.log(`삭제 완료: ${recording.filePath}`);
        } catch (error) {
          this.logger.error(`삭제 실패: ${recording.filePath}`, error.message);
        }
      }
    }

    this.logger.log('오래된 녹화 파일 삭제 완료');
  }
}
```

---

## Cron 표현식

```typescript
// 자주 쓰는 표현식
@Cron('0 3 * * *')              // 매일 03:00
@Cron('0 */6 * * *')            // 6시간마다
@Cron('*/30 * * * *')           // 30분마다
@Cron(CronExpression.EVERY_HOUR) // 매시간 (enum 사용)
```

---

## 인터벌 기반 스케줄링

```typescript
import { Interval } from '@nestjs/schedule';

@Interval(60000)  // 60초마다
async checkStreamHealth() {
  // MediaMTX API로 스트림 상태 확인
}
```

---

## 주의 사항

**다중 인스턴스 환경 (스케일 아웃)**

여러 NestJS 인스턴스가 실행 중이면 각 인스턴스가 동시에 cron을 실행한다.
파일 삭제가 중복 실행되면 에러가 발생한다.

해결 방법:
- Redis 기반 분산 락 (redis-lock)
- 단일 cron 전용 인스턴스로 분리
- DB 레벨 락 (`SELECT FOR UPDATE`)

---

## 서비스 등록

```typescript
// recording.module.ts
@Module({
  providers: [
    RecordingService,
    RecordingCleanupService,  // 추가
  ],
})
export class RecordingModule {}
```

`@Cron` 데코레이터가 붙은 메서드가 있는 클래스를 Provider로 등록하면 자동으로 스케줄러가 실행한다.
