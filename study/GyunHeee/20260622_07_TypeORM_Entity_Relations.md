# TypeORM에서 테이블 간 관계(1:N, FK) 엔티티 매핑

## 핵심 데코레이터

| 관계 | 데코레이터 |
|------|-----------|
| 1:N (부모 쪽) | `@OneToMany` |
| N:1 (자식 쪽, FK 보유) | `@ManyToOne` |
| N:M | `@ManyToMany` + `@JoinTable` |
| 1:1 | `@OneToOne` + `@JoinColumn` |

---

## 카메라 시스템 주요 관계 예시

### Camera (1) → Recording (N)

```typescript
// camera.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, OneToMany } from 'typeorm';
import { Recording } from '../recording/recording.entity';

@Entity('cameras')
export class Camera {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @Column()
  rtspUrl: string;

  @OneToMany(() => Recording, (recording) => recording.camera)
  recordings: Recording[];
}
```

```typescript
// recording.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, ManyToOne, JoinColumn, CreateDateColumn } from 'typeorm';
import { Camera } from '../camera/camera.entity';

@Entity('recordings')
export class Recording {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  filePath: string;

  @CreateDateColumn()
  startedAt: Date;

  @Column({ nullable: true })
  endedAt: Date;

  @ManyToOne(() => Camera, (camera) => camera.recordings, { onDelete: 'CASCADE' })
  @JoinColumn({ name: 'camera_id' })
  camera: Camera;

  @Column()
  camera_id: number;
}
```

---

### Channel (1) → Camera (N)

```typescript
// channel.entity.ts
@Entity('channels')
export class Channel {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;

  @OneToMany(() => Camera, (camera) => camera.channel)
  cameras: Camera[];
}
```

```typescript
// camera.entity.ts 에 추가
@ManyToOne(() => Channel, (channel) => channel.cameras)
@JoinColumn({ name: 'channel_id' })
channel: Channel;
```

---

## 자주 쓰는 옵션

```typescript
// onDelete: 부모 삭제 시 자식 처리
@ManyToOne(() => Camera, { onDelete: 'CASCADE' })   // 같이 삭제
@ManyToOne(() => Camera, { onDelete: 'SET NULL' })  // null로 변경

// eager: 관계 자동 로딩 (기본값 false, 성능 주의)
@ManyToOne(() => Camera, { eager: false })

// nullable: FK 컬럼 null 허용 여부
@ManyToOne(() => Camera, { nullable: false })
```

---

## 관계 조회 방법

```typescript
// camera.repository.ts
async findWithRecordings(cameraId: number) {
  return this.cameraRepository.findOne({
    where: { id: cameraId },
    relations: ['recordings'],  // 명시적으로 join
  });
}

// QueryBuilder 방식
async findCamerasWithChannel() {
  return this.cameraRepository
    .createQueryBuilder('camera')
    .leftJoinAndSelect('camera.channel', 'channel')
    .getMany();
}
```

---

## 주의: N+1 문제

```typescript
// 나쁜 예: 루프 안에서 각 카메라의 recordings를 별도 쿼리로 조회
for (const camera of cameras) {
  camera.recordings = await recordingRepo.findBy({ camera_id: camera.id });
}

// 좋은 예: 한 번에 join
const cameras = await cameraRepo.find({ relations: ['recordings'] });
```

관계 데이터를 루프에서 개별 조회하면 쿼리가 N+1개 발생한다. `relations` 옵션이나 QueryBuilder로 한 번에 조회한다.
