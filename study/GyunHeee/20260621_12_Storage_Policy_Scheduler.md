# storage_policies 테이블과 실제 파일 정리 스케줄러

## storage_policies 테이블 구조

```sql
CREATE TABLE storage_policies (
  id              INT PRIMARY KEY AUTO_INCREMENT,
  camera_id       INT NOT NULL,
  retention_days  INT NOT NULL DEFAULT 30,   -- 보관 기간 (일)
  max_size_gb     FLOAT,                     -- 최대 용량 제한 (선택)
  created_at      DATETIME DEFAULT NOW(),
  updated_at      DATETIME DEFAULT NOW() ON UPDATE NOW(),
  FOREIGN KEY (camera_id) REFERENCES cameras(id) ON DELETE CASCADE
);
```

카메라마다 다른 보관 정책을 설정할 수 있다.
예: 정문 카메라는 90일 보관, 주차장 카메라는 7일 보관.

---

## 실제 실행 주체: NestJS Cron Job

`@nestjs/schedule`의 `@Cron` 데코레이터로 매일 새벽 스케줄러가 정책을 읽고 파일을 삭제한다.

```typescript
// storage/storage-cleanup.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository, LessThan } from 'typeorm';
import { promises as fs } from 'fs';

@Injectable()
export class StorageCleanupService {
  private readonly logger = new Logger(StorageCleanupService.name);

  constructor(
    @InjectRepository(StoragePolicy)
    private readonly policyRepo: Repository<StoragePolicy>,
    @InjectRepository(RecordingSegment)
    private readonly segmentRepo: Repository<RecordingSegment>,
  ) {}

  @Cron(CronExpression.EVERY_DAY_AT_3AM)  // 매일 새벽 3시
  async cleanupExpiredSegments(): Promise<void> {
    this.logger.log('보관 정책 기반 파일 정리 시작');

    const policies = await this.policyRepo.find({ relations: ['camera'] });

    for (const policy of policies) {
      await this.cleanupByPolicy(policy);
    }

    this.logger.log('파일 정리 완료');
  }

  private async cleanupByPolicy(policy: StoragePolicy): Promise<void> {
    const expiryDate = new Date();
    expiryDate.setDate(expiryDate.getDate() - policy.retentionDays);

    // 만료된 세그먼트 조회
    const expiredSegments = await this.segmentRepo.find({
      where: {
        recording: { cameraId: policy.cameraId },
        createdAt: LessThan(expiryDate),
      },
      relations: ['recording'],
    });

    for (const segment of expiredSegments) {
      await this.deleteSegment(segment);
    }

    // 용량 제한도 적용
    if (policy.maxSizeGb) {
      await this.cleanupBySize(policy);
    }
  }

  private async deleteSegment(segment: RecordingSegment): Promise<void> {
    try {
      // 1. 파일 삭제
      await fs.unlink(segment.filePath);

      // 2. DB 레코드 삭제
      await this.segmentRepo.remove(segment);

      this.logger.debug(`삭제: ${segment.filePath}`);
    } catch (error) {
      if (error.code === 'ENOENT') {
        // 파일이 이미 없으면 DB 레코드만 정리
        await this.segmentRepo.remove(segment);
        this.logger.warn(`파일 없음, DB만 정리: ${segment.filePath}`);
      } else {
        this.logger.error(`삭제 실패: ${segment.filePath}`, error.message);
      }
    }
  }

  private async cleanupBySize(policy: StoragePolicy): Promise<void> {
    const maxBytes = policy.maxSizeGb * 1024 ** 3;

    // 카메라의 전체 세그먼트 용량 합산
    const result = await this.segmentRepo
      .createQueryBuilder('s')
      .select('SUM(s.fileSizeBytes)', 'total')
      .innerJoin('s.recording', 'r')
      .where('r.cameraId = :cameraId', { cameraId: policy.cameraId })
      .getRawOne();

    if (result.total > maxBytes) {
      // 오래된 것부터 삭제
      const oldest = await this.segmentRepo.find({
        where: { recording: { cameraId: policy.cameraId } },
        order: { createdAt: 'ASC' },
        take: 100,
      });
      for (const segment of oldest) {
        await this.deleteSegment(segment);
      }
    }
  }
}
```

---

## 빈 recording 정리

세그먼트가 모두 삭제된 recording 레코드는 별도로 정리한다.

```typescript
@Cron('30 3 * * *')  // 새벽 3시 30분 (segment 정리 후)
async cleanupEmptyRecordings(): Promise<void> {
  await this.recordingRepo
    .createQueryBuilder()
    .delete()
    .where(
      'id NOT IN (SELECT DISTINCT recording_id FROM recording_segments)',
    )
    .execute();
}
```

---

## 스케줄러 등록

```typescript
// storage.module.ts
@Module({
  providers: [StorageCleanupService],
  imports: [
    TypeOrmModule.forFeature([StoragePolicy, RecordingSegment, Recording]),
    ScheduleModule,
  ],
})
export class StorageModule {}
```

---

## 실행 흐름 요약

```
매일 새벽 3:00
    ↓
storage_policies 전체 조회
    ↓ (카메라별 정책 순회)
만료 세그먼트 조회 (createdAt < 오늘 - retention_days)
    ↓
HDD 파일 삭제 → DB 레코드 삭제
    ↓
용량 제한 초과 시 오래된 것부터 추가 삭제
    ↓
새벽 3:30
    ↓
세그먼트 없는 빈 recording 레코드 정리
```
