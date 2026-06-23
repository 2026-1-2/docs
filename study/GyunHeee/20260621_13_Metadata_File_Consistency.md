# 메타데이터(MySQL)와 실제 파일(HDD) 불일치 방지·복구

## 불일치 유형

| 유형 | 상황 | 설명 |
|------|------|------|
| Orphan File | HDD에 파일 있음, DB에 레코드 없음 | DB 저장 실패 후 파일만 남은 경우 |
| Broken Record | DB에 레코드 있음, HDD에 파일 없음 | 파일 삭제 후 DB 정리 안 된 경우 |
| Partial Write | 파일 있지만 손상됨 | 녹화 중 서버 강제 종료 |

---

## 방지 전략 1: 저장 순서 원칙

**파일 먼저, DB는 성공 확인 후** 원칙을 지킨다.

```typescript
// 나쁜 예: DB 먼저 저장 → 파일 쓰기 실패 시 broken record 발생
async saveSegment(filePath: string, recordingId: number) {
  await this.segmentRepo.save({ filePath, recordingId });  // DB 먼저
  await fs.copyFile(tempPath, filePath);  // 파일 쓰기 실패 가능
}

// 좋은 예: 파일 확인 후 DB 저장
async saveSegment(filePath: string, recordingId: number) {
  // 파일이 실제로 존재하고 크기가 0 이상인지 확인
  const stat = await fs.stat(filePath);
  if (stat.size === 0) throw new Error('빈 파일');

  // 파일 확인 후 DB 저장
  await this.segmentRepo.save({ filePath, recordingId, fileSizeBytes: stat.size });
}
```

---

## 방지 전략 2: chokidar 기반 파일 감지

파일이 실제로 생성된 이후 DB에 기록하는 구조.

```typescript
// file-watcher.service.ts
import * as chokidar from 'chokidar';

@Injectable()
export class FileWatcherService implements OnModuleInit {
  onModuleInit() {
    const watcher = chokidar.watch('/recordings/**/*.ts', {
      persistent: true,
      ignoreInitial: false,
      awaitWriteFinish: {
        stabilityThreshold: 500,  // 500ms 동안 크기 변화 없으면 완료로 판단
      },
    });

    watcher.on('add', (filePath) => this.onSegmentCreated(filePath));
  }

  private async onSegmentCreated(filePath: string) {
    // 파일이 실제로 존재할 때만 DB 저장
    const stat = await fs.stat(filePath);
    await this.segmentRepo.save({ filePath, fileSizeBytes: stat.size });
  }
}
```

파일이 완전히 쓰여진 후에 DB에 기록하므로 partial write 문제를 줄인다.

---

## 복구 전략: 정합성 검사 스케줄러

```typescript
// storage/consistency-check.service.ts
@Injectable()
export class ConsistencyCheckService {

  // 매주 일요일 새벽 4시
  @Cron('0 4 * * 0')
  async checkConsistency(): Promise<void> {
    await this.findBrokenRecords();
    await this.findOrphanFiles();
  }

  // DB에는 있지만 파일이 없는 레코드 탐지
  private async findBrokenRecords(): Promise<void> {
    const segments = await this.segmentRepo.find();

    for (const segment of segments) {
      try {
        await fs.access(segment.filePath);  // 파일 존재 확인
      } catch {
        this.logger.warn(`Broken record: ${segment.filePath}`);
        await this.segmentRepo.remove(segment);  // DB 레코드 제거
      }
    }
  }

  // HDD에는 있지만 DB에 없는 파일 탐지
  private async findOrphanFiles(): Promise<void> {
    const files = await this.getAllFilesOnDisk('/recordings');

    for (const filePath of files) {
      const exists = await this.segmentRepo.findOne({ where: { filePath } });
      if (!exists) {
        this.logger.warn(`Orphan file: ${filePath}`);
        // 정책에 따라: 삭제하거나 별도 보관 디렉토리로 이동
        await fs.rename(filePath, filePath.replace('/recordings/', '/orphans/'));
      }
    }
  }

  private async getAllFilesOnDisk(dir: string): Promise<string[]> {
    const { execSync } = await import('child_process');
    const output = execSync(`find ${dir} -name "*.ts" -type f`).toString();
    return output.trim().split('\n').filter(Boolean);
  }
}
```

---

## 녹화 상태 컬럼으로 partial write 감지

```sql
ALTER TABLE recordings ADD COLUMN status ENUM('recording', 'completed', 'failed') DEFAULT 'recording';
```

```typescript
// 서버 재시작 시 'recording' 상태인 레코드 처리
async onModuleInit() {
  const incompleteRecordings = await this.recordingRepo.find({
    where: { status: 'recording' },
  });

  for (const recording of incompleteRecordings) {
    // 서버가 재시작됐다는 건 녹화가 비정상 종료된 것
    await this.recordingRepo.update(recording.id, { status: 'failed' });
    this.logger.warn(`비정상 종료 감지: recording ${recording.id}`);
  }
}
```

---

## 요약

| 문제 | 방지 | 복구 |
|------|------|------|
| Broken record | 파일 존재 확인 후 DB 저장 | 주간 정합성 검사로 DB 레코드 제거 |
| Orphan file | chokidar로 파일 완료 후 DB 기록 | 고아 파일을 `/orphans/`로 격리 |
| Partial write | `awaitWriteFinish` 옵션 | `status: failed` 마킹 후 수동 검토 |
