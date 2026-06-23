# 환경별 설정값(ConfigModule) 분리 관리

## 목표

- 개발(dev)과 운영(prod) 환경의 설정값이 다름
- 코드에 하드코딩된 값이 없어야 함
- 환경변수 누락 시 빌드 단계에서 바로 잡힘

---

## 파일 구조

```
.env.development     # 로컬 개발용
.env.production      # 운영 서버용
.env.test            # 테스트용
src/config/
  app.config.ts      # 설정 스키마 정의
```

---

## .env 파일 예시

```env
# .env.development
NODE_ENV=development
PORT=3000

# DB
DB_HOST=localhost
DB_PORT=3306
DB_USERNAME=root
DB_PASSWORD=1234
DB_DATABASE=cctv_dev

# MediaMTX
MEDIAMTX_API_URL=http://localhost:9997
MEDIAMTX_TIMEOUT=5000
WEBHOOK_SECRET=dev-secret-key

# JWT
JWT_SECRET=dev-jwt-secret
JWT_EXPIRES_IN=1d
```

```env
# .env.production
NODE_ENV=production
PORT=3000

DB_HOST=prod-db-server
DB_PASSWORD=강한패스워드
MEDIAMTX_API_URL=http://mediamtx:9997
JWT_SECRET=프로덕션용_긴_랜덤_시크릿
```

---

## ConfigModule 설정

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,           // 모든 모듈에서 주입 없이 사용 가능
      envFilePath: `.env.${process.env.NODE_ENV || 'development'}`,
      validationSchema: Joi.object({   // 필수 변수 검증
        DB_HOST: Joi.string().required(),
        DB_PASSWORD: Joi.string().required(),
        JWT_SECRET: Joi.string().min(32).required(),
        MEDIAMTX_API_URL: Joi.string().uri().required(),
      }),
    }),
  ],
})
export class AppModule {}
```

---

## 타입 안전한 설정 접근 (Config Namespace)

```typescript
// config/app.config.ts
import { registerAs } from '@nestjs/config';

export const mediamtxConfig = registerAs('mediamtx', () => ({
  apiUrl: process.env.MEDIAMTX_API_URL,
  timeout: parseInt(process.env.MEDIAMTX_TIMEOUT, 10) || 5000,
  webhookSecret: process.env.WEBHOOK_SECRET,
}));
```

```typescript
// mediamtx.service.ts
import { ConfigService } from '@nestjs/config';

@Injectable()
export class MediaMtxService {
  constructor(private readonly configService: ConfigService) {}

  getApiUrl() {
    return this.configService.get<string>('mediamtx.apiUrl');
  }
}
```

---

## Docker Compose에서 환경별 주입

```yaml
# docker-compose.yml (개발)
services:
  nestjs:
    env_file:
      - .env.development

# docker-compose.prod.yml (운영)
services:
  nestjs:
    env_file:
      - .env.production
    # 또는 Docker secrets, Vault 등 사용
```

---

## 핵심 원칙

- `.env.*` 파일은 `.gitignore`에 추가 (`.env.example`만 커밋)
- 운영 환경 시크릿은 환경변수 파일이 아닌 Secret Manager(AWS Secrets Manager, Vault 등) 사용 권장
- Joi 검증으로 필수 값 누락을 앱 시작 시점에 잡기
