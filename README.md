# 📖 종프2 2팀 Wiki

> 드론 이착륙장(버티포트) CCTV 기반 감시 시스템 개발

안녕하세요! **엠엠피 과제** 의 모든 것을 기록하고 공유하는 위키입니다.  
이곳에서 스터디 목표, 진행 상황, 학습 자료, 회의록 등 모든 정보를 찾아볼 수 있습니다.

---

## 🗓️ 회의록

| 날짜 | 회의 |
| --- | --- |
| 2026.03.05 (목) | [회사 방문 회의 - 1차](./minutes/2026.03.05.%28%EB%AA%A9%29%20%ED%9A%8C%EC%82%AC%20%EB%B0%A9%EB%AC%B8%20%ED%9A%8C%EC%9D%98%20-%201%EC%B0%A8%20copy.md) |
| 2026.03.10 (화) | [수행계획 요구분석 발표 회의 - 3차](./minutes/2026.03.10.%28%ED%99%94%29%20%EC%88%98%ED%96%89%EA%B3%84%ED%9A%8D%20%EC%9A%94%EA%B5%AC%EB%B6%84%EC%84%9D%20%EB%B0%9C%ED%91%9C%20%ED%9A%8C%EC%9D%98%20-%203%EC%B0%A8.md) |
| 2026.03.20 (금) | [비대면 회의 - 4차](./minutes/2026.03.20.%28%EA%B8%88%29%20%EB%B9%84%EB%8C%80%EB%A9%B4%20%ED%9A%8C%EC%9D%98%20-%204%EC%B0%A8.md) |
| 2026.04.07 (화) | [회사 방문 회의 - 2차](./minutes/2026.04.07.%28%ED%99%94%29%20%ED%9A%8C%EC%82%AC%20%EB%B0%A9%EB%AC%B8%20%ED%9A%8C%EC%9D%98%20-%202%EC%B0%A8.md) |
| 2026.04.28 (화) | [비대면 회의 - 개발 중간 점검](./minutes/2026.04.28.%28%ED%99%94%29%20%EB%B9%84%EB%8C%80%EB%A9%B4%20%ED%9A%8C%EC%9D%98%20-%20%EA%B0%9C%EB%B0%9C%20%EC%A4%91%EA%B0%84%20%EC%A0%90%EA%B2%80.md) |
| 2026.05.12 (화) | [회사 방문 회의 - 3차](./minutes/2026.05.12.%28%ED%99%94%29%20%ED%9A%8C%EC%82%AC%20%EB%B0%A9%EB%AC%B8%20%ED%9A%8C%EC%9D%98%20-%203%EC%B0%A8.md) |
| 2026.06.02 (화) | [회사 방문 회의 - Demo Video 녹화](./minutes/2026.06.02.%28%ED%99%94%29%20%ED%9A%8C%EC%82%AC%20%EB%B0%A9%EB%AC%B8%20%ED%9A%8C%EC%9D%98%20-%20Demo%20Video%20%EB%85%B9%ED%99%94.md) |
| 2026.06.06 (토) | [비대면 회의 - SW 등록 및 성능 검증](./minutes/2026.06.06.%28%ED%86%A0%29%20%EB%B9%84%EB%8C%80%EB%A9%B4%20%ED%9A%8C%EC%9D%98%20-%20SW%20%EB%93%B1%EB%A1%9D%20%EB%B0%8F%20%EC%84%B1%EB%8A%A5%20%EA%B2%80%EC%A6%9D.md) |

> 새 회의록은 [Template.md](./minutes/Template.md) 를 복사해 작성하고, 위 표에 한 줄 추가합니다.

---

## 🧰 기술 스택

| 영역 | 기술 | 저장소 |
| --- | --- | --- |
| **Frontend** | React 19, TypeScript, Vite, Tailwind CSS 4, Zustand, TanStack Query, Dockview, hls.js | `client` |
| **Backend** | NestJS 11 (TypeScript), REST API, ONVIF PTZ 제어 | `mmp-server` |
| **영상 서버** | MediaMTX (RTSP → WebRTC/HLS 변환) | `media-server` |
| **녹화** | Python 3.12 + FFmpeg (RTSP → TS segment, codec copy), MySQL | `video-recorder` |
| **인프라** | Docker Compose, Nginx 리버스 프록시 | `infra` |
| **HW / 네트워크** | IP 카메라 (ONVIF PTZ), PoE 스위치, CAT.6 | - |

### 핵심 설계 원칙

- **백엔드는 영상 데이터를 직접 다루지 않는다** — MediaMTX / FFmpeg 에 위임, API·메타데이터만 관리
- **실시간 관제**: WebRTC (WHEP), 0.5초 이하 지연
- **녹화 재생**: HLS, 타임라인 탐색 지원
- **녹화 저장**: FFmpeg codec copy (무트랜스코딩, CPU 부하 최소)

```
IP Camera (RTSP) → MediaMTX → ┬→ WebRTC (실시간, client)
                              ├→ HLS    (녹화 재생, client)
                              └→ RTSP   (녹화 저장, video-recorder)
                          mmp-server (REST API / PTZ) ── client
```

---

## 📚 학습 인덱스

- **서형철** — [바로가기](./study/wjdqh6544/)
- **유지훈** — [바로가기](./study/wlgns12370/)
- **이견희** — [바로가기](./study/GyunHeee/)
- **이상민** — [바로가기](./study/lsmin3388/)
- **이준섭** — [바로가기](./study/SubJeeLee/)

---

## 🤝 협업 컨벤션

- [브랜치 / Issue / PR 컨벤션](./%EB%B8%8C%EB%9E%9C%EC%B9%98_Issue_PR_%EC%BB%A8%EB%B2%A4%EC%85%98.md)
- Issue / PR 템플릿은 [`.github`](./.github) 에 정식 파일로 등록되어 있습니다.

---

## 📂 문서 인덱스

- [QA 산출물](./qa) — 통합 테스트 계획·시나리오, 성능 측정 결과, 체크리스트
- [보안 문서](./security) — 위협 모델, 보안 점검 체크리스트
- [팀원별 기여 내역](./CONTRIBUTIONS.md)
