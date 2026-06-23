# RTSP 인증 정보를 DB에 어떻게 저장해야 하는가

## 평문 저장의 문제

```sql
-- 나쁜 예
INSERT INTO cameras (name, rtsp_url, rtsp_username, rtsp_password)
VALUES ('정문', 'rtsp://192.168.1.10/stream', 'admin', 'password123');
```

DB가 탈취되면 모든 카메라의 로그인 정보가 그대로 노출된다.
SQL Injection 취약점이 하나라도 있으면 즉시 전체 카메라 계정이 유출된다.

---

## 비밀번호는 단방향 해시인가 양방향 암호화인가

RTSP 비밀번호는 **복호화가 필요하다.** MediaMTX에 RTSP 연결 시 원본 비밀번호를 사용해야 하기 때문이다.

```
bcrypt 같은 단방향 해시 → 복호화 불가 → RTSP 연결에 사용 불가
AES 같은 양방향 암호화 → 복호화 가능 → RTSP 연결에 사용 가능
```

단방향 해시(bcrypt, argon2)는 로그인 비밀번호 검증용이다.
RTSP 비밀번호처럼 원본이 필요한 경우 **양방향 암호화(AES-256)**를 사용한다.

---

## AES-256 암호화 적용

### 암호화 키 관리

```env
# .env (git에 올리지 않음)
ENCRYPTION_KEY=32바이트_랜덤_16진수_문자열  # 256비트
```

암호화 키가 유출되면 암호화가 무의미해지므로 환경변수로 분리한다.

### NestJS에서 암호화/복호화 구현

```typescript
// crypto/encryption.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly key: Buffer;

  constructor(private readonly configService: ConfigService) {
    this.key = Buffer.from(configService.get('ENCRYPTION_KEY'), 'hex');
  }

  encrypt(plaintext: string): string {
    const iv = crypto.randomBytes(12);           // 매번 다른 IV
    const cipher = crypto.createCipheriv(this.algorithm, this.key, iv);

    const encrypted = Buffer.concat([
      cipher.update(plaintext, 'utf8'),
      cipher.final(),
    ]);
    const authTag = cipher.getAuthTag();

    // iv + authTag + encrypted를 하나의 문자열로 저장
    return Buffer.concat([iv, authTag, encrypted]).toString('base64');
  }

  decrypt(ciphertext: string): string {
    const data = Buffer.from(ciphertext, 'base64');
    const iv = data.slice(0, 12);
    const authTag = data.slice(12, 28);
    const encrypted = data.slice(28);

    const decipher = crypto.createDecipheriv(this.algorithm, this.key, iv);
    decipher.setAuthTag(authTag);

    return Buffer.concat([
      decipher.update(encrypted),
      decipher.final(),
    ]).toString('utf8');
  }
}
```

### 카메라 저장/조회 시 적용

```typescript
// camera.service.ts
async create(dto: CreateCameraDto): Promise<Camera> {
  const camera = this.cameraRepository.create({
    ...dto,
    rtspPassword: this.encryptionService.encrypt(dto.rtspPassword),
  });
  return this.cameraRepository.save(camera);
}

async getRtspUrl(camera: Camera): Promise<string> {
  const password = this.encryptionService.decrypt(camera.rtspPassword);
  return `rtsp://${camera.rtspUsername}:${password}@${camera.host}/stream`;
}
```

DB에는 암호화된 값이 저장되고, MediaMTX에 연결할 때만 복호화한다.

---

## DB 컬럼 설계

```typescript
// camera.entity.ts
@Entity('cameras')
export class Camera {
  @Column()
  rtspUsername: string;       // 평문 저장 가능 (username은 민감도 낮음)

  @Column({ select: false })  // 기본 조회에서 제외
  rtspPassword: string;       // AES-256 암호화 저장

  @Column()
  rtspUrl: string;            // 인증 정보 없는 URL만 저장
}
```

`select: false` 옵션으로 비밀번호 컬럼이 일반 조회에서 자동 제외된다.

---

## API 응답에서 비밀번호 제거

```typescript
// camera.controller.ts
@Get()
async findAll() {
  const cameras = await this.cameraService.findAll();
  return cameras.map(({ rtspPassword, ...rest }) => rest);  // 비밀번호 제거
}
```

클라이언트에게 비밀번호가 절대 반환되지 않도록 응답에서 제거한다.
