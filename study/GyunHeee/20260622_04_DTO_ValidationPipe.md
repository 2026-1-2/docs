# DTO와 ValidationPipe를 카메라 등록 API에 적용하는 방법

## 전체 흐름

```
HTTP 요청 → ValidationPipe → DTO 검증 → Controller → Service
```

요청 body가 DTO 조건을 만족하지 않으면 ValidationPipe가 400 에러를 반환한다.
Controller에 검증 로직이 들어가지 않는다.

---

## 패키지 설치

```bash
npm install class-validator class-transformer
```

---

## DTO 정의

```typescript
// dto/create-camera.dto.ts
import { IsString, IsNotEmpty, IsUrl, IsOptional, IsInt, Min } from 'class-validator';

export class CreateCameraDto {
  @IsString()
  @IsNotEmpty()
  name: string;

  @IsString()
  @IsNotEmpty()
  location: string;

  @IsUrl({ protocols: ['rtsp'], require_tld: false })
  rtspUrl: string;

  @IsOptional()
  @IsInt()
  @Min(1)
  channelId?: number;
}
```

---

## ValidationPipe 전역 등록

```typescript
// main.ts
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,       // DTO에 없는 필드 자동 제거
      forbidNonWhitelisted: true,  // 없는 필드 있으면 400 에러
      transform: true,       // 타입 자동 변환 (string → number 등)
    }),
  );

  await app.listen(3000);
}
```

---

## Controller 적용

```typescript
// camera.controller.ts
import { Body, Controller, Post } from '@nestjs/common';
import { CreateCameraDto } from './dto/create-camera.dto';
import { CameraService } from './camera.service';

@Controller('cameras')
export class CameraController {
  constructor(private readonly cameraService: CameraService) {}

  @Post()
  async createCamera(@Body() dto: CreateCameraDto) {
    return this.cameraService.create(dto);
  }
}
```

`@Body() dto: CreateCameraDto` 한 줄이면 ValidationPipe가 자동으로 검증한다.

---

## 유효성 검사 실패 시 응답 예시

```json
{
  "statusCode": 400,
  "message": [
    "rtspUrl must be a URL address",
    "name should not be empty"
  ],
  "error": "Bad Request"
}
```

---

## whitelist: true 가 중요한 이유

```json
// 요청 body
{
  "name": "정문",
  "rtspUrl": "rtsp://192.168.0.1/stream",
  "isAdmin": true   // DTO에 없는 필드
}
```

`whitelist: true`면 `isAdmin`은 조용히 제거된다.
`forbidNonWhitelisted: true`를 추가하면 400 에러를 반환한다.
의도치 않은 필드 주입을 막는 기본 보안 설정이다.
