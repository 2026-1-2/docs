# GitHub Actions CI 단계에서 자동화하는 테스트

## CI 파이프라인 목적

코드가 main(또는 PR 브랜치)에 올라올 때마다 자동으로 검증해서
"이 코드가 배포 가능한가"를 사람이 확인하기 전에 걸러내는 것이다.

---

## NestJS (mmp-server) CI 구성

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: mmp_test
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping -h localhost"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm

      - name: 의존성 설치
        run: npm ci

      - name: 타입 검사
        run: npm run type-check   # tsc --noEmit

      - name: 린트
        run: npm run lint

      - name: 단위 테스트 (Unit Test)
        run: npm run test -- --coverage
        env:
          NODE_ENV: test

      - name: E2E 테스트
        run: npm run test:e2e
        env:
          DB_HOST: 127.0.0.1
          DB_PORT: 3306
          DB_USERNAME: root
          DB_PASSWORD: test
          DB_DATABASE: mmp_test
          JWT_SECRET: test-secret-32chars-minimum
```

---

## 단위 테스트 vs E2E 테스트 범위

### 단위 테스트 (Unit Test)
- 서비스 메서드 단위로 검증
- 외부 의존성(DB, MediaMTX)은 Mock으로 대체
- 빠르게 실행 (수 초)

```typescript
// camera.service.spec.ts
describe('CameraService', () => {
  it('카메라 등록 시 MediaMTX에 path를 등록한다', async () => {
    const mockMediaMtx = { registerPath: jest.fn() };
    const service = new CameraService(mockRepo, mockMediaMtx);

    await service.create({ name: 'cam1', rtspUrl: 'rtsp://...' });

    expect(mockMediaMtx.registerPath).toHaveBeenCalledWith('cam1', 'rtsp://...');
  });
});
```

### E2E 테스트
- 실제 HTTP 요청으로 전체 흐름 검증
- 테스트용 MySQL 컨테이너(CI services)에 실제 연결
- MediaMTX는 Mock 서버로 대체 또는 스킵

```typescript
// camera.e2e-spec.ts
it('POST /cameras → 201 Created', () => {
  return request(app.getHttpServer())
    .post('/cameras')
    .set('Authorization', `Bearer ${testToken}`)
    .send({ name: 'cam1', rtspUrl: 'rtsp://192.168.0.1/stream' })
    .expect(201);
});
```

---

## React (client) CI 구성

```yaml
  - name: 빌드 검사
    run: npm run build    # TypeScript 컴파일 + 번들링

  - name: 단위 테스트
    run: npm run test -- --watchAll=false

  - name: 린트
    run: npm run lint
```

React E2E(Cypress/Playwright)는 서버 환경이 필요해서 별도 워크플로우로 분리하거나 생략하는 경우가 많다.

---

## CI 실패 시 동작

PR에 테스트 실패가 있으면 머지 버튼이 비활성화된다 (브랜치 보호 규칙 설정 시).
테스트 통과 없이는 main에 코드가 들어가지 않는다.
