# mmp-server API 테스트 커버리지

## 테스트 계층 구성

```
단위 테스트 (Unit Test)   → 서비스 메서드, 빠름, Mock 사용
통합 테스트 (Integration) → DB 연결, 실제 쿼리 검증
E2E 테스트               → HTTP 요청 → 응답 전체 흐름
수동 테스트 (Postman)     → 개발 중 빠른 확인, 팀 공유
```

---

## 단위 테스트 커버 범위

```typescript
// camera.service.spec.ts
describe('CameraService', () => {
  describe('create', () => {
    it('카메라 저장 후 MediaMTX에 path를 등록한다')
    it('MediaMTX 등록 실패 시 DB 롤백한다')
    it('중복 이름의 카메라는 등록 거부한다')
  })

  describe('delete', () => {
    it('MediaMTX path 삭제 후 DB 레코드를 삭제한다')
    it('존재하지 않는 카메라 삭제 시 404를 반환한다')
  })

  describe('findAll', () => {
    it('rtspPassword 필드는 응답에 포함되지 않는다')
  })
})
```

```typescript
// flyable-evaluator.service.spec.ts
describe('FlyableEvaluatorService', () => {
  it('풍속 10m/s 초과 시 is_flyable = false')
  it('모든 조건 정상 시 is_flyable = true')
  it('여러 조건 동시 위반 시 reason에 모두 포함된다')
})
```

---

## E2E 테스트 커버 범위

Jest + Supertest로 실제 HTTP 요청 테스트. 테스트용 MySQL에 실제 연결.

```typescript
// test/camera.e2e-spec.ts
describe('/cameras (E2E)', () => {

  describe('POST /cameras', () => {
    it('유효한 요청 → 201 + 생성된 카메라 반환')
    it('rtspUrl 형식 오류 → 400 + 검증 메시지')
    it('인증 토큰 없음 → 401')
    it('관리자 권한 없음 → 403')
    it('중복 이름 → 409')
  })

  describe('GET /cameras', () => {
    it('카메라 목록 반환 → 200')
    it('응답에 rtspPassword 필드 없음')
    it('인증 토큰 없음 → 401')
  })

  describe('DELETE /cameras/:id', () => {
    it('존재하는 카메라 삭제 → 204')
    it('존재하지 않는 id → 404')
    it('관리자 권한 없음 → 403')
  })
})
```

```typescript
// test/auth.e2e-spec.ts
describe('/auth (E2E)', () => {
  it('올바른 이메일/비밀번호 → 200 + accessToken 반환')
  it('잘못된 비밀번호 → 401')
  it('존재하지 않는 이메일 → 401')
  it('1분에 5회 초과 로그인 시도 → 429')  // Rate limit 테스트
})
```

---

## Postman 활용

### 팀 공유 컬렉션

팀 전체가 같은 Postman 컬렉션을 공유한다.
API 스펙이 바뀌면 컬렉션도 함께 업데이트한다.

```
MMP Server API (Postman Collection)
├── Auth
│   ├── POST /auth/login
│   └── POST /auth/refresh
├── Cameras
│   ├── GET /cameras
│   ├── POST /cameras
│   ├── PATCH /cameras/:id
│   └── DELETE /cameras/:id
├── Alerts
│   └── GET /alerts/stream (SSE)
└── Webhook (내부 테스트용)
    ├── POST /webhook/stream-down
    └── POST /webhook/stream-ready
```

### 환경변수로 토큰 자동 관리

```json
// Postman Environment: local
{
  "BASE_URL": "http://localhost:3000",
  "ACCESS_TOKEN": ""
}
```

```javascript
// POST /auth/login 의 Tests 탭
const data = pm.response.json();
pm.environment.set("ACCESS_TOKEN", data.accessToken);
```

로그인 후 토큰이 환경변수에 자동 저장되고, 이후 요청에서 `Bearer {{ACCESS_TOKEN}}`으로 사용된다.

---

## 테스트하지 않은 범위와 이유

| 범위 | 미적용 이유 |
|------|-----------|
| MediaMTX 실제 연동 E2E | MediaMTX 컨테이너 의존, CI 환경 구성 복잡 |
| SSE 연결 지속 테스트 | Jest에서 SSE 스트림 검증 어려움, 수동 확인 |
| 파일 시스템 정리 스케줄러 | 실행 시간이 길어 단위 테스트로 로직만 검증 |
| 부하 테스트 | CI에서 실행 부적합, 별도 환경에서 k6로 수행 |

---

## 테스트 실행 명령

```bash
# 단위 테스트 + 커버리지
npm run test -- --coverage

# E2E 테스트 (테스트 DB 필요)
npm run test:e2e

# 특정 파일만
npm run test -- camera.service.spec.ts

# watch 모드
npm run test -- --watch
```

---

## 커버리지 목표

```
Statements: 70% 이상
Branches:   60% 이상
Functions:  70% 이상
```

핵심 비즈니스 로직(카메라 등록/삭제, is_flyable 판단, 인증)은 100% 커버를 목표로 한다.
유틸리티, DTO, 엔티티 파일은 낮아도 무방하다.
