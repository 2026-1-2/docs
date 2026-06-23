# NestJS에서 외부 API 호출: HttpModule vs axios

## 선택지 비교

| | HttpModule (NestJS 내장) | axios 직접 사용 |
|--|------------------------|---------------|
| 주입 방식 | DI 컨테이너로 주입 | 직접 import |
| 테스트 | Mock 교체 용이 | 별도 mock 설정 필요 |
| 설정 공유 | 모듈 레벨 baseURL/headers | 인스턴스별 설정 |
| 의존성 | `@nestjs/axios` 설치 필요 | axios만 설치 |
| RxJS | Observable 반환 | Promise 반환 |

---

## HttpModule 사용법

```typescript
// mediamtx.module.ts
import { HttpModule } from '@nestjs/axios';

@Module({
  imports: [
    HttpModule.register({
      baseURL: 'http://mediamtx:9997',
      timeout: 5000,
    }),
  ],
  providers: [MediaMtxService],
})
export class MediaMtxModule {}
```

```typescript
// mediamtx.service.ts
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';

@Injectable()
export class MediaMtxService {
  constructor(private readonly httpService: HttpService) {}

  async registerPath(name: string, rtspUrl: string) {
    const response = await firstValueFrom(
      this.httpService.post(`/v3/config/paths/add/${name}`, {
        source: rtspUrl,
      }),
    );
    return response.data;
  }
}
```

---

## axios 직접 사용법

```typescript
// mediamtx.service.ts
import axios from 'axios';

@Injectable()
export class MediaMtxService {
  private readonly client = axios.create({
    baseURL: 'http://mediamtx:9997',
    timeout: 5000,
  });

  async registerPath(name: string, rtspUrl: string) {
    const { data } = await this.client.post(`/v3/config/paths/add/${name}`, {
      source: rtspUrl,
    });
    return data;
  }
}
```

---

## MediaMTX 호출에 적합한 선택: HttpModule

**이유**

1. **테스트 용이성**: HttpService를 Mock으로 교체하면 MediaMTX 없이도 서비스 테스트 가능
2. **ConfigModule 연동**: baseURL을 환경변수로 주입하기 쉬움
3. **NestJS 철학 일관성**: DI 컨테이너 안에서 관리

```typescript
HttpModule.registerAsync({
  useFactory: (config: ConfigService) => ({
    baseURL: config.get('MEDIAMTX_API_URL'),
    timeout: config.get('MEDIAMTX_TIMEOUT'),
  }),
  inject: [ConfigService],
}),
```

**axios 직접 사용이 나은 경우**: 단순 스크립트, NestJS 외부 코드, RxJS가 불필요한 상황.

---

## 결론

NestJS 프로젝트에서 MediaMTX REST API 호출은 **HttpModule**이 적절하다.
테스트와 설정 관리가 DI 체계 안으로 들어오기 때문이다.
