# MediaMTX API 명세 어디서 확인하는가

## 공식 문서

```
https://bluenviron.github.io/mediamtx/
```

Control API 레퍼런스 섹션에서 엔드포인트 목록 확인 가능.

---

## OpenAPI YAML 파일 — 가장 정확한 소스

MediaMTX GitHub 저장소에 OpenAPI 명세 파일이 포함되어 있다.

```
https://github.com/bluenviron/mediamtx
└── apidocs/
    └── openapi.yaml     ← 여기
```

이 파일을 Swagger UI나 Stoplight에 붙여넣으면 인터랙티브 문서로 볼 수 있다.

### 로컬에서 보는 법
```bash
# MediaMTX 서버 실행 후 기본으로 Swagger UI 제공
http://localhost:9997/swagger
```

MediaMTX가 실행 중이면 `/swagger` 경로에서 바로 확인할 수 있다.

---

## 주요 API 엔드포인트 요약

### Path 관리 (카메라 등록/수정/삭제)
```
GET    /v3/config/paths/list            # 등록된 path 전체 조회
GET    /v3/config/paths/get/{name}      # 특정 path 조회
POST   /v3/config/paths/add/{name}      # path 추가 (카메라 등록)
PATCH  /v3/config/paths/patch/{name}    # path 수정
DELETE /v3/config/paths/delete/{name}   # path 삭제
```

### 스트림 상태 조회 (헬스체크)
```
GET    /v3/paths/list                   # 현재 활성 스트림 목록
GET    /v3/paths/get/{name}             # 특정 스트림 상태
```

### 세션 조회
```
GET    /v3/rtspconns/list               # RTSP 연결 목록
GET    /v3/hls/muxers/list              # HLS 세션 목록
GET    /v3/webrtcsessions/list          # WebRTC 세션 목록
```

---

## 헷갈리기 쉬운 포인트

- **포트**: API 기본 포트는 `9997`, 스트리밍 포트(RTSP)는 `8554`로 다르다.
- **`/v3/config/paths`** vs **`/v3/paths`**: config는 설정 관리, paths는 현재 상태 조회다. 카메라 등록은 config, 헬스체크는 paths.
- **API 서버 활성화**: `mediamtx.yml`에서 `api: yes` 설정이 있어야 API가 열린다.

```yaml
# mediamtx.yml
api: yes
apiAddress: :9997
```
