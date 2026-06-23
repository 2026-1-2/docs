# MediaMTX Webhook 스푸핑 방어 장치

## 위협 시나리오

MediaMTX webhook 엔드포인트(`POST /webhook/stream-down`)는 인증이 없으면 누구나 호출할 수 있다.

```
공격자 → POST https://your-domain.com/webhook/stream-down {"name": "cam1"}
NestJS  → 가짜 이벤트를 실제로 처리 (알람 생성, 녹화 중단 등)
```

위조된 요청으로 가짜 알람을 대량 발생시키거나, 정상 스트림을 "다운"으로 처리하게 할 수 있다.

---

## 방어 계층 1: 네트워크 격리 (1차 방어)

MediaMTX와 NestJS가 같은 Docker 네트워크에 있고, webhook 포트가 외부에 노출되지 않으면 외부에서 접근 자체가 불가능하다.

```yaml
# docker-compose.yml
services:
  mmp-server:
    networks:
      - mmp-network
    # ports 없음 → 외부에서 /webhook/* 에 직접 접근 불가
    # Caddy가 /webhook/* 경로를 프록시하지 않으면 외부 접근 차단

  mediamtx:
    networks:
      - mmp-network  # 같은 내부 네트워크에서만 webhook 호출
```

Caddyfile에서 `/webhook/*` 경로를 `handle`하지 않으면 외부에서 도달할 수 없다.

---

## 방어 계층 2: Shared Secret 헤더 검증 (2차 방어)

네트워크 격리가 뚫리거나, 같은 네트워크의 다른 컨테이너가 위조 요청을 보내는 경우를 대비한다.

### MediaMTX에서 시크릿 헤더 포함

```yaml
# mediamtx.yml
paths:
  cam1:
    runOnNotReady: >
      curl -s -X POST http://mmp-server:3000/webhook/stream-down
      -H "Content-Type: application/json"
      -H "X-Webhook-Secret: ${WEBHOOK_SECRET}"
      -d '{"name":"cam1","event":"not_ready"}'
```

### NestJS Guard에서 검증

```typescript
// guards/webhook-secret.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as crypto from 'crypto';

@Injectable()
export class WebhookSecretGuard implements CanActivate {
  constructor(private readonly configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const receivedSecret = request.headers['x-webhook-secret'];
    const expectedSecret = this.configService.get<string>('WEBHOOK_SECRET');

    // 타이밍 공격 방지: 단순 문자열 비교 대신 crypto.timingSafeEqual 사용
    if (!receivedSecret) throw new UnauthorizedException();

    const received = Buffer.from(receivedSecret);
    const expected = Buffer.from(expectedSecret);

    if (received.length !== expected.length ||
        !crypto.timingSafeEqual(received, expected)) {
      throw new UnauthorizedException('Invalid webhook secret');
    }

    return true;
  }
}
```

```typescript
// webhook.controller.ts
@UseGuards(WebhookSecretGuard)
@Controller('webhook')
export class WebhookController {
  @Post('stream-down')
  async handleStreamDown(@Body() body: WebhookDto) {
    await this.alertService.pushAlert({ type: 'STREAM_DOWN', name: body.name });
  }
}
```

---

## 방어 계층 3: 요청 출처 IP 검증 (선택적 3차 방어)

```typescript
// guards/mediamtx-ip.guard.ts
@Injectable()
export class MediaMtxIpGuard implements CanActivate {
  private readonly allowedIp: string;

  constructor(private readonly configService: ConfigService) {
    this.allowedIp = this.configService.get('MEDIAMTX_CONTAINER_IP');
  }

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const clientIp = request.ip || request.connection.remoteAddress;

    if (clientIp !== this.allowedIp) {
      throw new UnauthorizedException('Webhook from unauthorized IP');
    }
    return true;
  }
}
```

Docker 내부 IP는 재시작 시 바뀔 수 있으므로, 컨테이너 이름 기반 DNS 필터링이 더 안정적이다.

---

## 방어 계층 4: Webhook DTO 검증

위조된 요청이 통과하더라도 예상치 못한 페이로드를 거른다.

```typescript
// dto/webhook.dto.ts
import { IsString, IsNotEmpty, IsIn } from 'class-validator';

export class WebhookDto {
  @IsString()
  @IsNotEmpty()
  name: string;  // 등록된 카메라 이름만 허용

  @IsIn(['ready', 'not_ready', 'read', 'unread'])
  event: string;
}
```

---

## 방어 계층 요약

```
외부 인터넷
    ↓
[Caddy] /webhook/* 경로 미노출 → 1차 차단
    ↓ (내부 네트워크에서의 요청만 통과)
[NestJS WebhookSecretGuard] X-Webhook-Secret 검증 → 2차 차단
    ↓
[ValidationPipe] DTO 검증 → 3차 차단
    ↓
실제 비즈니스 로직 처리
```

네트워크 격리 + 시크릿 헤더 검증 두 계층만으로도 실질적인 보호가 된다.
`crypto.timingSafeEqual`을 사용하는 이유는 단순 `===` 비교가 타이밍 공격에 취약하기 때문이다.
