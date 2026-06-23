# RTSP 카메라는 내부망, 외부 관리자는 HTTPS로 접근하는 구조

## 핵심 아이디어

카메라(RTSP)와 MediaMTX 사이는 내부망으로 연결한다.
외부 관리자는 RTSP를 직접 받지 않고, MediaMTX가 변환한 HLS/WebRTC를 HTTPS로 받는다.

```
[IP 카메라] ── RTSP (사내 내부망) ──→ [MediaMTX] ── HLS/WebRTC ──→ [Caddy HTTPS] ──→ [외부 관리자]
```

프로토콜 변환이 MediaMTX 안에서 일어나고, 외부에는 HTTP(S)만 노출된다.

---

## 전체 네트워크 구조

```
[사내 내부망 192.168.x.x]
┌─────────────────────────────────────────┐
│  IP 카메라들 (192.168.1.10~50)           │
│       ↓ RTSP (포트 554)                  │
│  MediaMTX 컨테이너 (192.168.1.100)       │
│       ↓ HLS (포트 8888)                  │
│  Caddy 컨테이너 (192.168.1.100)          │
│       ↓ HTTPS (포트 443)                 │
└─────────────────────────────────────────┘
         ↕ 공유기/방화벽 (443만 허용)
[외부 인터넷]
    외부 관리자 브라우저
```

**방화벽 규칙**: 외부에서 443(HTTPS)만 허용. RTSP 554, 8888 등은 차단.

---

## 포트별 접근 제어

| 포트 | 프로토콜 | 내부 접근 | 외부 접근 | 용도 |
|------|---------|---------|---------|------|
| 554 | RTSP | 허용 | **차단** | 카메라 → MediaMTX |
| 8554 | RTSP | 허용 | **차단** | (MediaMTX RTSP 재배포) |
| 8888 | HLS | 허용 | **차단** | MediaMTX → Caddy (내부) |
| 443 | HTTPS | 허용 | **허용** | Caddy → 외부 관리자 |
| 3000 | HTTP | 허용 | **차단** | NestJS (내부만) |
| 3306 | MySQL | 허용 | **차단** | DB (내부만) |

---

## docker-compose.yml 포트 설정

```yaml
services:
  mediamtx:
    ports:
      - "8554:8554"   # RTSP: 내부망 카메라만 접근 (방화벽으로 외부 차단)
    # 8888(HLS)은 ports 없음 → Caddy가 내부 네트워크로 접근

  caddy:
    ports:
      - "80:80"       # HTTP-01 challenge용 (또는 HTTP→HTTPS 리다이렉트)
      - "443:443"     # 외부에 유일하게 노출되는 포트
```

---

## 외부 관리자의 영상 시청 흐름

```
1. 외부 관리자 → HTTPS://your-domain.com (React 앱 로드)
2. React → HTTPS://your-domain.com/api/cameras (카메라 목록 요청)
3. NestJS → React: [{id:1, name:"정문", hlsUrl:"/streams/cam1/index.m3u8"}, ...]
4. React → HTTPS://your-domain.com/streams/cam1/index.m3u8 (HLS 재생 시작)
5. Caddy → mediamtx:8888/cam1/index.m3u8 (내부망으로 프록시)
6. 세그먼트 파일 반복 요청으로 영상 재생
```

외부 관리자는 HTTPS 하나만 사용. 내부 포트, RTSP 주소, DB 정보는 전혀 모른다.

---

## 카메라 추가 시 흐름

```
외부 관리자 → POST /api/cameras {rtspUrl: "rtsp://192.168.1.20/stream"}
NestJS → POST mediamtx:9997/v3/config/paths/add/cam2 (내부망 API 호출)
MediaMTX → 192.168.1.20:554 RTSP 연결 (내부망)
관리자 → /streams/cam2/index.m3u8 으로 바로 시청 가능
```

외부 관리자는 RTSP URL을 "등록"만 하고, 실제 RTSP 연결은 서버(내부망)가 담당한다.

---

## 이 구조의 보안적 이점

1. **카메라 RTSP 미노출**: 외부에서 카메라에 직접 접근 불가
2. **단일 진입점**: 443 포트 하나만 관리하면 됨
3. **인증 계층 통합**: 모든 요청이 Caddy → NestJS를 거치므로 JWT 인증 적용 가능
4. **내부망 격리**: DB, MediaMTX API(9997) 등 민감한 서비스가 외부에 노출되지 않음
