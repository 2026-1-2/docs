# Exception Filter로 MediaMTX API 호출 실패 핸들링

## 기본 개념

Exception Filter는 서비스에서 던진 예외를 잡아서 일관된 HTTP 응답으로 변환한다.
MediaMTX가 응답하지 않거나 에러를 반환할 때, 이걸 그대로 클라이언트에 노출하지 않고 의미 있는 메시지로 바꾼다.

---

## MediaMTX 전용 예외 클래스 정의

```typescript
// exceptions/mediamtx.exception.ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class MediaMtxException extends HttpException {
  constructor(message: string, status: HttpStatus = HttpStatus.BAD_GATEWAY) {
    super({ message, source: 'MediaMTX' }, status);
  }
}

export class MediaMtxPathNotFoundException extends MediaMtxException {
  constructor(pathName: string) {
    super(`MediaMTX path not found: ${pathName}`, HttpStatus.NOT_FOUND);
  }
}
```

---

## 서비스에서 예외 변환

```typescript
// mediamtx.service.ts
import { HttpService } from '@nestjs/axios';
import { firstValueFrom } from 'rxjs';
import { AxiosError } from 'axios';

@Injectable()
export class MediaMtxService {
  constructor(private readonly httpService: HttpService) {}

  async registerPath(name: string, rtspUrl: string): Promise<void> {
    try {
      await firstValueFrom(
        this.httpService.post(`/v3/config/paths/add/${name}`, { source: rtspUrl }),
      );
    } catch (error) {
      if (error instanceof AxiosError) {
        if (error.code === 'ECONNREFUSED') {
          throw new MediaMtxException('MediaMTX 서버에 연결할 수 없습니다', HttpStatus.SERVICE_UNAVAILABLE);
        }
        if (error.response?.status === 404) {
          throw new MediaMtxPathNotFoundException(name);
        }
        throw new MediaMtxException(`MediaMTX 오류: ${error.response?.data?.error}`);
      }
      throw error;
    }
  }
}
```

---

## Exception Filter 구현

```typescript
// filters/mediamtx-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpException } from '@nestjs/common';
import { Response } from 'express';
import { MediaMtxException } from '../exceptions/mediamtx.exception';

@Catch(MediaMtxException)
export class MediaMtxExceptionFilter implements ExceptionFilter {
  catch(exception: MediaMtxException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const status = exception.getStatus();
    const body = exception.getResponse() as any;

    response.status(status).json({
      statusCode: status,
      message: body.message,
      source: body.source,
      timestamp: new Date().toISOString(),
    });
  }
}
```

---

## 전역 또는 컨트롤러 단위 등록

```typescript
// 전역 등록 (main.ts)
app.useGlobalFilters(new MediaMtxExceptionFilter());

// 컨트롤러 단위 등록
@UseFilters(MediaMtxExceptionFilter)
@Controller('cameras')
export class CameraController {}
```

---

## 결과: 클라이언트가 받는 응답

MediaMTX 연결 실패 시:
```json
{
  "statusCode": 503,
  "message": "MediaMTX 서버에 연결할 수 없습니다",
  "source": "MediaMTX",
  "timestamp": "2026-06-22T10:30:00.000Z"
}
```

MediaMTX 내부 에러 원인이 그대로 노출되지 않고, 클라이언트에게 의미 있는 메시지가 전달된다.
