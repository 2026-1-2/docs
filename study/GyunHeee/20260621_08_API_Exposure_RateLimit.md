# REST API 외부 노출 범위와 Rate Limiting

## 외부에 노출되는 API 범위

전체 아키텍처에서 외부(인터넷)에 노출되는 진입점은 Caddy 443 포트 하나다.
NestJS(mmp-server)는 Caddy 뒤에 있고, `/api/*` 경로만 프록시된다.

### 공개 엔드포인트 (인증 없음)

```
POST /api/auth/login       ← 로그인 (토큰 발급)
POST /api/auth/refresh     ← 토큰 갱신
```

### 인증 필요 엔드포인트 (JWT Guard)

```
GET    /api/cameras          ← 카메라 목록 (일반 사용자)
POST   /api/cameras          ← 카메라 등록 (관리자)
PATCH  /api/cameras/:id      ← 카메라 수정 (관리자)
DELETE /api/cameras/:id      ← 카메라 삭제 (관리자)
GET    /api/alerts/stream    ← SSE 알람 스트림 (인증 필요)
GET    /api/recordings       ← 녹화 목록 조회
```

### 외부 노출 차단 엔드포인트

```
POST /webhook/*              ← MediaMTX webhook (내부 전용, IP 제한)
GET  /api/health             ← 헬스체크 (Caddy 내부 사용)
```

---

## Rate Limiting 적용

### 패키지 설치

```bash
npm install @nestjs/throttler
```

### 전역 설정

```typescript
// app.module.ts
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,    // 1초
        limit: 10,    // 10회
      },
      {
        name: 'long',
        ttl: 60000,   // 1분
        limit: 100,   // 100회
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,  // 전역 적용
    },
  ],
})
export class AppModule {}
```

### 로그인 엔드포인트 강화 (브루트포스 방어)

```typescript
// auth.controller.ts
import { Throttle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {
  @Throttle({ short: { ttl: 60000, limit: 5 } })  // 1분에 5회만 허용
  @Post('login')
  async login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }
}
```

로그인 시도를 1분에 5회로 제한해서 브루트포스 공격을 차단한다.

### Rate Limit 초과 시 응답

```json
{
  "statusCode": 429,
  "message": "ThrottlerException: Too Many Requests"
}
```

---

## Caddy 레벨 Rate Limiting

NestJS 이전에 Caddy에서 먼저 거르는 것도 가능하다.

```caddyfile
your-domain.com {
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 100
            window 1m
        }
    }

    handle /api/* {
        reverse_proxy mmp-server:3000
    }
}
```

Caddy의 `rate_limit` 디렉티브는 별도 플러그인이 필요하다.
기본 Caddy에는 포함되어 있지 않으므로 NestJS ThrottlerGuard만으로도 충분하다.

---

## CORS 설정

외부 도메인에서의 API 요청 제한.

```typescript
// main.ts
app.enableCors({
  origin: ['https://your-domain.com'],  // 허용 도메인 명시
  methods: ['GET', 'POST', 'PATCH', 'DELETE'],
  credentials: true,
});
```

`origin: '*'`은 개발 환경에서만 사용한다. 운영 환경에서는 허용 도메인을 명시한다.

---

## 보안 헤더 추가

```typescript
// main.ts
import helmet from 'helmet';

app.use(helmet());  // XSS, clickjacking, MIME sniffing 등 기본 보안 헤더 추가
```

Helmet이 추가하는 주요 헤더:
- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `Strict-Transport-Security` (HTTPS 강제)
- `Content-Security-Policy`
