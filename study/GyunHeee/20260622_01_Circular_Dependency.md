# NestJS 모듈 간 순환 의존성(Circular Dependency) 해결

## 언제 발생하는가

```
CameraModule → StreamModule (StreamService를 주입받음)
StreamModule → CameraModule (CameraService를 주입받음)
```

두 모듈이 서로를 imports 배열에 올리면 NestJS가 의존성 그래프를 해석하지 못하고 에러를 낸다.

---

## 해결 방법 1: forwardRef (가장 흔한 방법)

```typescript
// camera.module.ts
@Module({
  imports: [forwardRef(() => StreamModule)],
  providers: [CameraService],
  exports: [CameraService],
})
export class CameraModule {}

// stream.module.ts
@Module({
  imports: [forwardRef(() => CameraModule)],
  providers: [StreamService],
  exports: [StreamService],
})
export class StreamModule {}
```

서비스 주입 시에도 동일하게 적용한다.

```typescript
// stream.service.ts
constructor(
  @Inject(forwardRef(() => CameraService))
  private readonly cameraService: CameraService,
) {}
```

`forwardRef`는 모듈/클래스 참조를 지연 평가해서 순환을 끊는다.

---

## 해결 방법 2: 의존성 방향을 단방향으로 재설계 (근본 해결)

순환이 생겼다는 건 설계에서 책임이 뒤섞였다는 신호다.

```
변경 전: CameraModule ↔ StreamModule (상호 의존)
변경 후: CameraModule → StreamModule (단방향)
         StreamModule은 CameraService를 모름
```

StreamService가 CameraService를 직접 부르는 대신, 이벤트(EventEmitter)나 공통 모듈(SharedModule)로 소통하게 리팩토링한다.

---

## 해결 방법 3: SharedModule 분리

두 서비스가 공통으로 쓰는 로직을 별도 모듈로 빼낸다.

```
CameraModule  StreamModule
      ↓              ↓
    SharedModule (공통 로직)
```

---

## 권장 순서

1. `forwardRef`로 일단 빌드 통과 → 기능 구현 완료
2. 설계 검토 후 의존성 방향 재설계 → 근본 해결

프로젝트 초기에는 forwardRef로 막고, 안정화 단계에서 구조를 개선하는 게 현실적이다.
