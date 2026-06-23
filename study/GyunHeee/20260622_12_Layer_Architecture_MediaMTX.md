# 컨트롤러-서비스-레포지토리 계층에서 MediaMTX 제어 로직의 위치

## 계층 구조 복습

```
Controller   ← HTTP 요청/응답 처리만 담당
    ↓
Service      ← 비즈니스 로직, 외부 시스템 연동
    ↓
Repository   ← DB CRUD만 담당
```

---

## MediaMTX 제어 로직은 Service 계층에 둔다

### 이유

Controller는 HTTP 요청을 받아서 Service를 호출하고 결과를 반환하는 역할만 한다.
Repository는 DB와 대화하는 역할만 한다.
MediaMTX REST API 호출은 **외부 시스템과의 연동** — 이건 Service의 책임이다.

---

## 잘못된 구조 예시

```typescript
// 나쁜 예: Controller에서 MediaMTX 직접 호출
@Post()
async createCamera(@Body() dto: CreateCameraDto) {
  const camera = await this.cameraService.create(dto);
  // Controller가 외부 API를 직접 부름 — 잘못된 위치
  await this.httpService.post(`/v3/config/paths/add/${camera.name}`, {...});
  return camera;
}
```

---

## 올바른 구조

### MediaMTX 전용 서비스 분리

```typescript
// mediamtx/mediamtx.service.ts  ← MediaMTX API 호출 전담
@Injectable()
export class MediaMtxService {
  constructor(private readonly httpService: HttpService) {}

  async registerPath(name: string, rtspUrl: string): Promise<void> {
    await firstValueFrom(
      this.httpService.post(`/v3/config/paths/add/${name}`, { source: rtspUrl }),
    );
  }

  async deletePath(name: string): Promise<void> {
    await firstValueFrom(
      this.httpService.delete(`/v3/config/paths/delete/${name}`),
    );
  }
}
```

### CameraService에서 조합

```typescript
// camera/camera.service.ts  ← 비즈니스 로직 조합
@Injectable()
export class CameraService {
  constructor(
    private readonly cameraRepository: CameraRepository,
    private readonly mediaMtxService: MediaMtxService,
  ) {}

  async create(dto: CreateCameraDto): Promise<Camera> {
    // 1. DB에 저장
    const camera = await this.cameraRepository.save(dto);

    // 2. MediaMTX에 등록 (같은 Service 계층에서 조합)
    await this.mediaMtxService.registerPath(camera.name, camera.rtspUrl);

    return camera;
  }

  async delete(id: number): Promise<void> {
    const camera = await this.cameraRepository.findOneOrFail({ where: { id } });

    // 1. MediaMTX에서 제거
    await this.mediaMtxService.deletePath(camera.name);

    // 2. DB에서 삭제
    await this.cameraRepository.remove(camera);
  }
}
```

### Controller는 위임만 한다

```typescript
// camera/camera.controller.ts
@Controller('cameras')
export class CameraController {
  constructor(private readonly cameraService: CameraService) {}

  @Post()
  async create(@Body() dto: CreateCameraDto) {
    return this.cameraService.create(dto);  // 한 줄
  }

  @Delete(':id')
  async delete(@Param('id') id: number) {
    return this.cameraService.delete(id);   // 한 줄
  }
}
```

---

## 계층별 책임 정리

| 계층 | 담당 | MediaMTX 관련 |
|------|------|--------------|
| Controller | HTTP 요청 파싱, 응답 반환 | 관여하지 않음 |
| CameraService | 카메라 등록/삭제 비즈니스 로직 | MediaMtxService를 호출 |
| MediaMtxService | MediaMTX REST API 호출 | 직접 HTTP 요청 |
| CameraRepository | DB CRUD | 관여하지 않음 |

---

## 이렇게 분리하면 얻는 것

- **테스트 용이성**: MediaMtxService를 Mock으로 교체하면 CameraService를 DB 없이 테스트 가능
- **교체 용이성**: MediaMTX가 다른 미디어 서버로 바뀌어도 MediaMtxService만 수정
- **관심사 분리**: 각 계층이 자기 역할에만 집중
