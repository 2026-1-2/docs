# TypeORM vs Prisma 비교 및 프로젝트 선택 기준

## ORM(Object Relational Mapping)이란?

ORM은 코드에서 객체를 다루듯이 데이터베이스를 사용할 수 있게 해주는 기술이다.

NestJS에서 대표적으로 많이 사용하는 ORM은 다음 두 가지이다.

- TypeORM
- Prisma

둘 다 MySQL, PostgreSQL 같은 관계형 데이터베이스와 연결할 수 있지만 개발 방식과 철학이 다르다.

---

# 1. TypeORM

## 개념

TypeORM은 TypeScript 기반 ORM이며, NestJS에서 오래전부터 많이 사용되어 온 ORM이다.

Spring Boot의 JPA와 구조가 매우 비슷하다.

```ts
@Entity()
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  name: string;
}
```

Entity 클래스를 만들면 데이터베이스 테이블과 매핑된다.

---

## 특징

- Entity 기반 개발
- Repository 패턴 사용
- NestJS 공식 지원
- Spring JPA와 유사한 구조

---

## 장점

### 1. Spring JPA와 매우 유사함

```ts
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>,
  ) {}

  findAll() {
    return this.userRepository.find();
  }
}
```

Spring Boot 경험이 있다면 매우 익숙하다.

---

### 2. 객체지향 설계에 적합

관계 설정을 Entity 내부에서 관리한다.

```ts
@OneToMany(() => Post, post => post.user)
posts: Post[];
```

---

### 3. NestJS와 연동이 자연스러움

```ts
TypeOrmModule.forRoot({
  type: 'mysql',
  host: 'localhost',
  port: 3306,
  username: 'root',
  password: 'password',
  database: 'test',
  entities: [User],
  synchronize: true,
});
```

---

## 단점

### 1. 타입 안정성이 Prisma보다 약함

```ts
relations: ['posts']
```

문자열 기반 relation 설정이라 오타를 컴파일 단계에서 잡기 어렵다.

---

### 2. synchronize 옵션이 위험함

```ts
synchronize: true
```

운영 환경에서는 데이터 손실 위험이 있다.

---

### 3. 복잡한 쿼리가 길어짐

```ts
const users = await this.userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.posts', 'post')
  .where('user.name = :name', { name })
  .getMany();
```

---

# 2. Prisma

## 개념

Prisma는 TypeScript 중심 ORM이다.

Entity 클래스를 만드는 대신 `schema.prisma` 파일로 데이터베이스 구조를 관리한다.

```prisma
model User {
  id    Int    @id @default(autoincrement())
  name  String
  posts Post[]
}
```

---

## 특징

- schema.prisma 기반 개발
- 강력한 타입 안정성
- 자동완성 지원
- Prisma Client 자동 생성
- migration 관리가 쉬움

---

## 장점

### 1. 타입 안정성이 매우 강력함

```ts
const user = await this.prisma.user.findUnique({
  where: { id: 1 },
  include: {
    posts: true,
  },
});
```

relation까지 타입 추론이 가능하다.

---

### 2. 자동완성이 매우 좋음

```ts
this.prisma.user.findMany({
  where: {
    name: {
      contains: 'kim',
    },
  },
});
```

필드명과 조건문 자동완성이 지원된다.

---

### 3. migration 관리가 직관적임

```bash
npx prisma migrate dev --name init
```

DB 변경 이력을 명확하게 관리할 수 있다.

---

### 4. Prisma Studio 제공

```bash
npx prisma studio
```

브라우저에서 DB 데이터를 GUI로 확인 가능하다.

---

### 5. 개발 속도가 빠름

CRUD 작성 속도가 매우 빠르다.

---

## 단점

### 1. Spring JPA 방식과 다름

Entity 중심 개발이 아니라 schema 중심 개발이다.

---

### 2. 매우 복잡한 SQL은 raw query가 필요할 수 있음

```ts
await this.prisma.$queryRaw`
  SELECT * FROM users WHERE name = ${name}
`;
```

---

# 3. TypeORM vs Prisma 비교

| 항목 | TypeORM | Prisma |
|---|---|---|
| 개발 방식 | Entity 기반 | Schema 기반 |
| Spring JPA 유사성 | 매우 높음 | 낮음 |
| 타입 안정성 | 보통 | 매우 강함 |
| 자동완성 | 보통 | 매우 좋음 |
| migration | 다소 복잡 | 매우 직관적 |
| 생산성 | 보통 | 높음 |
| 유지보수 | 보통 | 좋음 |
| 팀 프로젝트 | 좋음 | 매우 좋음 |
| 러닝 커브 | 낮음 | 보통 |
| NestJS 사용성 | 좋음 | 매우 좋음 |

---

# 4. 현재 프로젝트 기준 분석

## 프로젝트 특징

현재 프로젝트는 다음 기능들을 포함한다.

- 카메라 관리
- RTSP 스트리밍 관리
- 녹화 파일 메타데이터 관리
- 객체 인식 결과 저장
- 이벤트 로그 저장
- 관리자 대시보드 API
- AI 분석 결과 저장

즉, relation이 많아지는 구조이다.

```txt
Camera
 ├── Recording
 ├── DetectionEvent
 └── StreamStatus
```

---

# 5. 왜 Prisma를 추천하는가?

## 1. 타입 안정성이 매우 중요함

```ts
const camera = await this.prisma.camera.findUnique({
  where: { id: cameraId },
  include: {
    recordings: true,
    detectionEvents: true,
  },
});
```

relation까지 타입 추론이 가능하다.

실수를 줄이고 유지보수가 쉬워진다.

---

## 2. 팀 프로젝트에서 schema 관리가 쉬움

```prisma
model Camera {
  id        Int      @id @default(autoincrement())
  name      String
  rtspUrl   String
  createdAt DateTime @default(now())

  recordings Recording[]
}
```

DB 구조를 한 파일에서 바로 파악할 수 있다.

---

## 3. migration 관리가 쉬움

```bash
npx prisma migrate dev --name add_detection_event
```

기능 추가가 많은 프로젝트에서 매우 편하다.

---

## 4. 개발 속도가 빠름

자동완성과 타입 지원 덕분에 CRUD 개발 속도가 빠르다.

---

## 5. NestJS와 궁합이 좋음

```ts
@Injectable()
export class CameraService {
  constructor(private readonly prisma: PrismaService) {}

  async findAll() {
    return this.prisma.camera.findMany();
  }
}
```

구조가 단순하고 유지보수가 쉽다.

---

# 6. TypeORM을 선택해도 되는 경우

다음 조건이라면 TypeORM도 좋은 선택이다.

- Spring Boot JPA 방식이 익숙할 때
- Entity 중심 설계를 선호할 때
- Repository 패턴을 강하게 사용하고 싶을 때
- 기존 프로젝트가 TypeORM 기반일 때

---

# 7. 최종 결론

## 현재 프로젝트에서는 Prisma를 추천한다.

### 이유

- 타입 안정성이 뛰어남
- 자동완성이 강력함
- migration 관리가 쉬움
- 팀 프로젝트 유지보수에 유리함
- relation이 많은 구조에 적합함
- 개발 속도가 빠름

---

# 8. 추천 구조

```txt
src
 ├── prisma
 │    ├── prisma.module.ts
 │    └── prisma.service.ts
 │
 ├── cameras
 │    ├── cameras.controller.ts
 │    ├── cameras.service.ts
 │    └── dto
 │
 ├── recordings
 │    ├── recordings.controller.ts
 │    ├── recordings.service.ts
 │    └── dto
 │
 ├── detection-events
 │    ├── detection-events.controller.ts
 │    ├── detection-events.service.ts
 │    └── dto
 │
 └── app.module.ts
```

---

# 9. 예시 Prisma Schema

```prisma
model Camera {
  id        Int      @id @default(autoincrement())
  name      String
  rtspUrl   String
  location  String?
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  recordings      Recording[]
  detectionEvents DetectionEvent[]
}

model Recording {
  id        Int      @id @default(autoincrement())
  cameraId  Int
  filePath  String
  startedAt DateTime
  endedAt   DateTime?
  createdAt DateTime @default(now())

  camera Camera @relation(fields: [cameraId], references: [id])
}

model DetectionEvent {
  id          Int      @id @default(autoincrement())
  cameraId    Int
  objectType  String
  confidence  Float
  imageUrl    String?
  detectedAt  DateTime
  createdAt   DateTime @default(now())

  camera Camera @relation(fields: [cameraId], references: [id])
}
```

---

# 10. 한 줄 요약

| ORM | 추천도 |
|---|---|
| TypeORM | Spring JPA 느낌이 강하고 익숙함 |
| Prisma | 현재 NestJS + TypeScript 프로젝트에서는 가장 추천 |
