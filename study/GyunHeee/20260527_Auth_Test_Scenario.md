# 인증 & 권한 테스트 시나리오

---

## TC-AUTH-01: Public 엔드포인트 — 토큰 없이 접근 가능

| # | 요청 | 기대 응답 |
|---|---|---|
| 1 | `GET /health` | `200` |
| 2 | `GET /version` | `200` |
| 3 | `POST /auth/login` (올바른 자격증명) | `200` |
| 4 | `POST /auth/login` (잘못된 비밀번호) | `401` |
| 5 | `POST /auth/register` | `201` |
| 6 | `POST /streams/mediamtx-auth` (유효하지 않은 token) | `401` |

---

## TC-AUTH-02: 토큰 없음 → 401

| # | 요청 | 기대 응답 |
|---|---|---|
| 1 | `GET /cameras` | `401` |
| 2 | `POST /cameras` | `401` |
| 3 | `DELETE /cameras/1` | `401` |
| 4 | `GET /streams/sessions` | `401` |
| 5 | `GET /streams/active` | `401` |
| 6 | `POST /auth/logout` | `401` |

---

## TC-AUTH-03: VIEWER 역할 경계

| # | 요청 | 기대 응답 | 이유 |
|---|---|---|---|
| 1 | `GET /cameras` | `200` | VIEWER 허용 |
| 2 | `GET /cameras/:id` | `200` | VIEWER 허용 |
| 3 | `GET /cameras/:id/snapshot` | `200` | VIEWER 허용 |
| 4 | `POST /cameras` | `403` | ADMIN 전용 |
| 5 | `PATCH /cameras/:id` | `403` | ADMIN 전용 |
| 6 | `DELETE /cameras/:id` | `403` | ADMIN 전용 |
| 7 | `POST /cameras/health-check-all` | `403` | OPERATOR 이상 |
| 8 | `GET /streams/sessions` | `403` | ADMIN 전용 |
| 9 | `GET /streams/active` | `403` | OPERATOR 이상 |

---

## TC-AUTH-04: OPERATOR 역할 경계

| # | 요청 | 기대 응답 | 이유 |
|---|---|---|---|
| 1 | `GET /cameras` | `200` | VIEWER 이상 허용 |
| 2 | `POST /cameras/health-check-all` | `200` | OPERATOR 허용 |
| 3 | `GET /streams/active` | `200` | OPERATOR 허용 |
| 4 | `POST /cameras` | `403` | ADMIN 전용 |
| 5 | `DELETE /cameras/:id` | `403` | ADMIN 전용 |
| 6 | `GET /streams/sessions` | `403` | ADMIN 전용 |

---

## TC-AUTH-05: ADMIN 역할 — 모든 API 접근 가능

| # | 요청 | 기대 응답 | 이유 |
|---|---|---|---|
| 1 | `GET /cameras` | `200` | — |
| 2 | `GET /streams/sessions` | `200` | — |
| 3 | `GET /streams/active` | `200` | — |
| 4 | `DELETE /cameras/999` | `404` | 접근은 허용, 카메라 없음 |
| 5 | `POST /cameras/999/health-check` | `404` | 접근은 허용, 카메라 없음 |

---

## TC-AUTH-06: SSE 인증

| # | 요청 | 기대 응답 |
|---|---|---|
| 1 | `GET /sse/events` (token 없음) | `401` |
| 2 | `GET /sse/events?token=invalid` | `401` |
| 3 | `GET /sse/events?token={유효한 JWT}` | `200`, `Content-Type: text/event-stream` |

---

## TC-AUTH-07: 토큰 갱신 / 로그아웃 흐름

| # | 요청 | 기대 응답 |
|---|---|---|
| 1 | `POST /auth/login` → access_token + refresh_token 발급 확인 | `200` |
| 2 | `POST /auth/refresh` (유효한 refresh_token) → 새 access_token 발급 | `200` |
| 3 | `POST /auth/refresh` (만료/유효하지 않은 refresh_token) | `401` |
| 4 | `POST /auth/logout` | `200` |
| 5 | 로그아웃 후 동일 refresh_token으로 재발급 시도 | `401` |

---
