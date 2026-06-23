# MediaMTX Webhook 엔드포인트 보호 방법

## 문제: webhook 엔드포인트는 외부에서 POST를 보낸다

MediaMTX가 NestJS에게 webhook을 보낸다.
이 엔드포인트를 열어두면 **아무나 POST를 보내서 가짜 이벤트를 주입**할 수 있다.
일반적인 JWT Guard를 달면 MediaMTX가 토큰을 갖고 있지 않으므로 동작하지 않는다.

---

## 방법 1: Shared Secret (가장 실용적)

NestJS와 MediaMTX 양쪽에 동일한 시크릿 키를 설정하고,
webhook 요청 헤더에 포함시켜 검증한다.

```typescript
// webhook-secret.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class WebhookSecretGuard implements CanActivate {
  constructor(private readonly configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const secret = request.headers['x-webhook-secret'];

    if (secret !== this.configService.get('WEBHOOK_SECRET')) {
      throw new UnauthorizedException('Invalid webhook secret');
    }
    return true;
  }
}
```

```typescript
// webhook.controller.ts
@Controller('webhook')
export class WebhookController {
  @UseGuards(WebhookSecretGuard)
  @Post('stream-down')
  async handleStreamDown(@Body() body: any) {
    // ...
  }
}
```

MediaMTX 설정에서 헤더를 추가한다.

```yaml
# mediamtx.yml
paths:
  cam1:
    runOnNotReady: >
      curl -X POST http://nestjs:3000/webhook/stream-down
      -H "x-webhook-secret: ${WEBHOOK_SECRET}"
      -H "Content-Type: application/json"
      -d '{"name":"cam1"}'
```

---

## 방법 2: IP 화이트리스트

MediaMTX 컨테이너의 IP만 허용한다.

```typescript
@Injectable()
export class IpWhitelistGuard implements CanActivate {
  private readonly allowedIps = ['172.18.0.5']; // MediaMTX 컨테이너 IP

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    const ip = request.ip;

    if (!this.allowedIps.includes(ip)) {
      throw new UnauthorizedException();
    }
    return true;
  }
}
```

Docker 내부 네트워크라면 외부에서 애초에 접근이 안 되므로 IP 화이트리스트로 충분한 경우가 많다.

---

## 방법 3: 내부 네트워크만 허용 (Docker Compose)

가장 간단한 방법. webhook 포트를 외부에 노출하지 않는다.

```yaml
# docker-compose.yml
services:
  nestjs:
    # ports를 열지 않거나, 내부 네트워크만 사용
    networks:
      - internal

  mediamtx:
    networks:
      - internal

networks:
  internal:
    internal: true  # 외부 인터넷 접근 차단
```

MediaMTX와 NestJS가 같은 Docker 네트워크에 있으면, 외부에서 webhook 엔드포인트에 접근할 수 없다.

---

## 권장 조합

```
내부 네트워크 격리 (Docker) + Shared Secret 헤더 검증
```

네트워크 레벨에서 1차 차단, 애플리케이션 레벨에서 2차 검증.
