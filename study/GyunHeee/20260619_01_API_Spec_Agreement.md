# 5명 팀원 간 API 스펙 합의 방식

## 왜 인터페이스 합의가 먼저인가

백엔드와 프론트엔드가 동시에 개발하려면 "무엇을 주고받을지"부터 정해야 한다.
스펙 없이 각자 구현하면 나중에 연동 시점에 불일치가 터진다.

---

## 합의 절차

### 1단계: 기능 단위로 API 목록 초안 작성

팀 전체가 모여 "어떤 화면에서 어떤 데이터가 필요한가"를 역으로 추적해서 API 목록을 뽑는다.

```
화면: 카메라 목록 페이지
  → 필요 데이터: 카메라 이름, 위치, 스트림 URL, 현재 상태
  → API: GET /cameras

화면: 카메라 등록 모달
  → 필요 데이터: 이름, 위치, RTSP URL 입력
  → API: POST /cameras

화면: 실시간 알람 패널
  → 필요 데이터: 이벤트 타입, 발생 시각, 카메라 이름
  → API: GET /alerts/stream (SSE)
```

프론트엔드 팀원이 "이 데이터 필요하다"고 요구하고, 백엔드 팀원이 "이렇게 줄 수 있다"고 조율한다.

### 2단계: Request/Response 형태 문서화

```markdown
## POST /cameras

**Request**
```json
{
  "name": "정문",
  "location": "A동 정문",
  "rtspUrl": "rtsp://192.168.1.10/stream",
  "channelId": 1
}
```

**Response 201**
```json
{
  "id": 42,
  "name": "정문",
  "location": "A동 정문",
  "status": "inactive",
  "createdAt": "2026-06-19T09:00:00.000Z"
}
```

**Response 400**
```json
{
  "statusCode": 400,
  "message": ["rtspUrl must be a URL address"]
}
```
```

응답 필드 이름, 타입, 날짜 포맷(ISO 8601 등)까지 명시한다.

### 3단계: 스펙 문서를 단일 소스로 관리

스펙을 팀 공유 문서(Notion, GitHub Wiki, docs 레포)에 올리고 변경 시 PR로 리뷰한다.
"구두로 합의했다"는 나중에 기억이 달라지므로 반드시 문서로 남긴다.

---

## 개발 중 스펙 변경 시 프로세스

```
변경 필요 발생
    ↓
변경 요청자가 스펙 문서 PR 올림
    ↓
영향받는 팀원(프론트/백엔드) 리뷰 + 승인
    ↓
스펙 문서 머지 후 각자 구현 변경
```

스펙 변경을 PR 없이 구현에만 반영하면 다른 팀원이 모르는 채로 개발이 엇갈린다.

---

## 프론트엔드가 백엔드 완료 전에 개발하는 방법: MSW

백엔드 API가 아직 없어도 프론트엔드 개발이 가능하다.

```typescript
// mocks/handlers.ts (MSW - Mock Service Worker)
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('/api/cameras', () => {
    return HttpResponse.json([
      { id: 1, name: '정문', status: 'active' },
      { id: 2, name: '주차장', status: 'inactive' },
    ]);
  }),
];
```

스펙 문서를 보고 목업 데이터를 만들어서 프론트엔드 개발을 진행한다.
백엔드가 완료되면 MSW만 끄면 실제 API로 전환된다.

---

## 역할별 책임

| 역할 | 스펙 관련 책임 |
|------|--------------|
| 백엔드 | 구현 가능한 구조 제안, 응답 필드 정의 |
| 프론트엔드 | 필요한 데이터 요구, UI 관점에서 검토 |
| 인프라 | 인증 방식(JWT), CORS, 도메인 구조 확인 |
| 전체 | 스펙 변경 PR 리뷰 참여 |
