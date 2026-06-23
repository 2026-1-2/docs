## 실시간 영상 모니터링 시퀀스

시스템 부팅부터 클라이언트의 실시간 재생, 그리고 장애 발생 시 복구까지의 흐름은 다음과 같다.

### 1. 서버 시작 (Server Startup)

NestJS 서버가 기동되면서 MediaMTX 컨테이너와의 연결 상태를 확인하고, 등록된 카메라 목록을 DB에서 조회해 초기화한다.

### 2. RTSP 연결 (RTSP Connection)

NestJS는 카메라 메타데이터(IP, 인증 정보 등)를 기반으로 MediaMTX에 스트림 경로 설정을 요청한다. MediaMTX는 이 설정을 바탕으로 HIKVISION PTZ 카메라에 RTSP 연결을 직접 수립한다. 이 시점부터 영상 바이트 데이터의 흐름은 MediaMTX의 책임 영역으로 들어가며, NestJS는 더 이상 영상 데이터 경로에 개입하지 않는다.

### 3. HLS 세그먼트 생성 (HLS Segmentation)

MediaMTX는 수신한 RTSP 스트림을 내부적으로 HLS 세그먼트(.m3u8 + .ts)로 변환하여 지속적으로 생성한다. 이 과정은 NestJS의 개입 없이 MediaMTX 단독으로 처리된다.

### 4. 클라이언트 재생 (Client Playback)

React 프론트엔드는 NestJS를 거치지 않고 MediaMTX에 직접 접근하여 HLS(hls.js) 또는 WebRTC(WHEP)로 스트림을 요청하고 재생한다. 이 직접 연결 구조는 NestJS 서버를 영상 데이터 부하로부터 분리시키는 핵심 설계 포인트다.

### 5. 장애 복구 (Failure Recovery)

카메라 연결이 끊기거나 스트림 처리 중 오류가 발생하면, MediaMTX는 이를 폴링이 아닌 **웹훅(webhook)** 방식으로 NestJS에 즉시 통지한다. NestJS는 해당 알림을 받아 카메라 상태를 DB에 갱신하고, 필요 시 재연결을 트리거하거나 사용자에게 SSE로 장애 상태를 전파한다.

---

### Mermaid 코드

\`\`\`mermaid
sequenceDiagram
    participant Cam as PTZ Camera
    participant MTX as MediaMTX
    participant API as NestJS (mmp-server)
    participant Client as React Client

    API->>MTX: 서버 시작 시 스트림 경로 설정 요청
    MTX->>Cam: RTSP 연결 수립
    Cam-->>MTX: RTSP 영상 스트림 전송
    MTX->>MTX: HLS 세그먼트 생성 (.m3u8/.ts)
    Client->>MTX: HLS/WebRTC 스트림 요청
    MTX-->>Client: 영상 스트림 응답 (직접 재생)

    Note over Cam,MTX: 장애 발생 (연결 끊김 등)
    MTX->>API: Webhook으로 장애 알림
    API->>API: 카메라 상태 갱신 (DB)
    API->>Client: SSE로 장애 상태 전파
    API->>MTX: 재연결 트리거 (필요 시)
\`\`\`

<img width="6356" height="8191" alt="RTSP to HLS Streaming-2026-06-23-052124" src="https://github.com/user-attachments/assets/8d3328aa-7c81-479d-a81b-f8dc8fc9121d" />
