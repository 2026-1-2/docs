# OnModuleInit vs OnApplicationBootstrap 차이

## 실행 시점 비교

```
NestJS 부팅 순서:

1. 모듈 인스턴스화
2. 의존성 주입 완료
3. OnModuleInit 실행       ← 각 모듈이 준비되는 즉시
4. 모든 모듈 부팅 완료
5. OnApplicationBootstrap  ← 애플리케이션 전체가 준비된 후
6. HTTP 서버 listen 시작
```

---

## OnModuleInit

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';

@Injectable()
export class CameraService implements OnModuleInit {
  async onModuleInit() {
    // 이 서비스가 속한 모듈의 의존성 주입이 완료된 직후 실행
    await this.initializeCameraChannels();
  }
}
```

**특징**
- 해당 모듈의 Provider가 모두 준비된 시점에 실행
- 다른 모듈이 아직 초기화 중일 수 있음
- 모듈 단위의 초기화 작업에 적합

---

## OnApplicationBootstrap

```typescript
import { Injectable, OnApplicationBootstrap } from '@nestjs/common';

@Injectable()
export class AppService implements OnApplicationBootstrap {
  async onApplicationBootstrap() {
    // 전체 앱이 준비된 후 실행 (HTTP 서버 열리기 직전)
    await this.registerAllWebhooks();
  }
}
```

**특징**
- 모든 모듈의 OnModuleInit이 완료된 후 실행
- 애플리케이션 전체 상태에 의존하는 작업에 적합
- 외부 시스템 연동, 전역 이벤트 등록 등

---

## 카메라 채널 초기화를 OnModuleInit에서 하는 이유

카메라 채널 초기화는 **CameraModule 내부 리소스만 있으면 된다.**

```typescript
@Injectable()
export class CameraService implements OnModuleInit {
  constructor(
    private readonly cameraRepository: CameraRepository,
    private readonly mediaMtxService: MediaMtxService,
  ) {}

  async onModuleInit() {
    const cameras = await this.cameraRepository.findAll();
    for (const camera of cameras) {
      await this.mediaMtxService.registerPath(camera);
    }
  }
}
```

- DB에서 카메라 목록 조회 → CameraRepository (같은 모듈)
- MediaMTX에 path 등록 → MediaMtxService (같은 모듈 또는 주입 완료)

다른 모듈(예: AlertModule, AuthModule)이 준비될 때까지 기다릴 필요가 없다.

**OnApplicationBootstrap을 쓰면 생기는 문제**: 모든 모듈이 뜰 때까지 카메라 채널이 등록되지 않는다. 부팅 시간이 길어지고, 초기화 실패 시 원인 추적이 어렵다.

**결론**: 모듈 단위로 독립적인 초기화는 OnModuleInit, 전체 앱 조율이 필요한 초기화는 OnApplicationBootstrap.
