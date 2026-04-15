# TypeScript 타입 시스템 핵심 정리

> **작성일**: 2026-04-15  
> **작성자**: lsmin3388  
> **프로젝트**: 버티포트 CCTV 감시 시스템

---

## 1. 개요

버티포트 프로젝트는 프론트엔드(React)와 백엔드(NestJS) 모두 TypeScript로 작성한다. 타입 시스템을 올바르게 활용하면 런타임 에러를 컴파일 타임에 잡을 수 있고, API 계약을 코드 수준에서 보장할 수 있다.

---

## 2. 기본 타입

```typescript
const cameraName: string = "Hikvision-DS-2CD2143G2";
const streamPort: number = 8554;
const isActive: boolean = true;
const cameraIds: number[] = [1, 2, 3, 4];
const resolution: [number, number] = [1920, 1080];  // 튜플

enum CameraStatus {
  ACTIVE = "ACTIVE", INACTIVE = "INACTIVE",
  MAINTENANCE = "MAINTENANCE", ERROR = "ERROR",
}
```

> 문자열 Enum은 디버깅이 직관적이다. 숫자 Enum은 Prisma 매핑에서 주의 필요.

---

## 3. 인터페이스 vs 타입 별칭

| 기능 | `interface` | `type` |
|---|---|---|
| 객체 형태 정의 | O | O |
| 선언 병합 | O | X |
| 유니온/교차 타입 | X | O |

```typescript
interface Camera {
  id: number; name: string; ipAddress: string;
  status: CameraStatus; rtspUrl: string; createdAt: Date;
}
type StreamType = "rtsp" | "hls" | "webrtc";

interface PTZCamera extends Camera {
  panRange: [number, number]; tiltRange: [number, number]; zoomLevel: number;
}
```

**컨벤션**: DTO/엔티티는 `interface`, 유니온/유틸리티 조합은 `type`.

---

## 4. 제네릭

```typescript
interface ApiResponse<T> { success: boolean; data: T; message: string; }
interface PaginatedResponse<T> { items: T[]; total: number; page: number; limit: number; }

async function fetchApi<T>(endpoint: string): Promise<ApiResponse<T>> {
  const response = await fetch(`/api/${endpoint}`);
  return response.json();
}

function findById<T extends { id: number }>(items: T[], id: number): T | undefined {
  return items.find((item) => item.id === id);
}
```

---

## 5. 유틸리티 타입

| 유틸리티 타입 | 설명 | 프로젝트 사용 예시 |
|---|---|---|
| `Partial<T>` | 모든 프로퍼티 선택적 | 카메라 업데이트 DTO |
| `Pick<T, K>` | 특정 프로퍼티 선택 | 목록 표시용 요약 타입 |
| `Omit<T, K>` | 특정 프로퍼티 제외 | 생성 DTO (id 제외) |
| `Record<K, V>` | 키-값 매핑 | 상태별 카메라 그룹 |

```typescript
type CreateCameraDto = Omit<Camera, "id" | "createdAt">;
type UpdateCameraDto = Partial<Pick<Camera, "name" | "ipAddress" | "status">>;
type CameraSummary = Pick<Camera, "id" | "name" | "status">;
type CamerasByStatus = Record<CameraStatus, Camera[]>;
```

---

## 6. 타입 가드

```typescript
interface RTSPStream { protocol: "rtsp"; rtspUrl: string; port: number; }
interface HLSStream  { protocol: "hls"; playlistUrl: string; }
type Stream = RTSPStream | HLSStream;

function isRTSPStream(stream: Stream): stream is RTSPStream {
  return stream.protocol === "rtsp";
}

function processStream(stream: Stream) {
  if (isRTSPStream(stream)) {
    console.log(`RTSP: ${stream.rtspUrl}:${stream.port}`);
  } else {
    console.log(`HLS: ${stream.playlistUrl}`);
  }
}
```

---

## 7. 유니온 타입과 교차 타입

```typescript
type LogLevel = "info" | "warn" | "error" | "debug";
type EventType = "motion_detected" | "camera_offline" | "stream_started";
interface Timestamped { createdAt: Date; updatedAt: Date; }
interface SoftDeletable { deletedAt: Date | null; isDeleted: boolean; }
type CameraEntity = Camera & Timestamped & SoftDeletable;
```

---

## 8. NestJS DTO 타입 활용

```typescript
import { IsString, IsIP, IsEnum, IsOptional, IsInt, Min, Max } from "class-validator";
import { ApiProperty } from "@nestjs/swagger";

export class CreateCameraDto {
  @ApiProperty({ example: "버티포트-A-CAM01" })
  @IsString()
  name: string;

  @ApiProperty({ example: "192.168.1.64" })
  @IsIP()
  ipAddress: string;

  @ApiProperty({ example: 554 })
  @IsInt() @Min(1) @Max(65535)
  rtspPort: number;

  @IsEnum(CameraStatus) @IsOptional()
  status?: CameraStatus;
}
```

> `PartialType(CreateCameraDto)`으로 UpdateCameraDto를 자동 생성할 수 있다.

---

## 9. React 컴포넌트 Props 타입

```typescript
import { FC, ReactNode } from "react";

interface CameraCardProps {
  camera: Camera;
  isSelected: boolean;
  onSelect: (id: number) => void;
  children?: ReactNode;
}

const CameraCard: FC<CameraCardProps> = ({ camera, isSelected, onSelect }) => (
  <div onClick={() => onSelect(camera.id)} className={isSelected ? "selected" : ""}>
    <h3>{camera.name}</h3>
    <span>{camera.status}</span>
  </div>
);

interface DataTableProps<T> {
  data: T[];
  columns: Array<{ key: keyof T; label: string }>;
  onRowClick?: (row: T) => void;
}
```

---

## 10. tsconfig.json 주요 옵션

```jsonc
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "baseUrl": ".",
    "paths": { "@/*": ["src/*"] },
    "esModuleInterop": true,
    "experimentalDecorators": true,
    "emitDecoratorMetadata": true,
    "skipLibCheck": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

| 옵션 | 설명 | 권장값 |
|---|---|---|
| `strict` | 전체 strict 모드 | `true` |
| `strictNullChecks` | null 안전성 보장 | `true` |
| `experimentalDecorators` | NestJS 데코레이터 | `true` (백엔드) |
| `paths` | 절대 경로 별칭 | 프로젝트 구조에 맞게 |

---

## 참고 자료

- [TypeScript 공식 Handbook](https://www.typescriptlang.org/docs/handbook/)
- [NestJS - Custom providers](https://docs.nestjs.com/fundamentals/custom-providers)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/)
