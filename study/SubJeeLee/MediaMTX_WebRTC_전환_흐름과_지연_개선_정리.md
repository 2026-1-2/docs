# MediaMTX WebRTC 전환 흐름과 지연 개선 정리

## 1. 먼저 결론부터

이전 방식은 **NestJS가 RTSP 영상을 FFmpeg로 HLS 파일로 변환한 뒤 프론트가 hls.js로 재생**하는 구조였다.

변경 후 방식은 **mediaMTX가 RTSP 카메라 스트림을 직접 받아 WebRTC로 중계하고, 프론트는 WebRTC URL로 mediaMTX에 직접 연결**하는 구조이다.

핵심 차이는 다음과 같다.

```text
이전 HLS 방식
RTSP Camera
  -> NestJS
  -> FFmpeg
  -> .ts segment 생성
  -> playlist.m3u8 갱신
  -> client hls.js 재생

변경 WebRTC 방식
RTSP Camera
  -> mediaMTX
  -> WebRTC WHEP
  -> client RTCPeerConnection 재생
```

HLS는 영상을 파일 조각으로 만든 뒤 재생하기 때문에 지연이 누적된다.

WebRTC는 파일 조각을 만들지 않고 브라우저와 실시간 연결을 맺어 미디어 패킷을 바로 전달하므로 지연이 크게 줄어든다.

---

## 2. 이전 구조: NestJS 내부 FFmpeg + HLS

이전 구조에서는 NestJS 서버가 RTSP 카메라 주소를 직접 읽고 FFmpeg 프로세스를 실행했다.

```text
RTSP Camera
  -> mmp-server
  -> FFmpeg 실행
  -> hls/ch101/playlist.m3u8 생성
  -> hls/ch101/seg00001.ts 생성
  -> GET /streams/ch101/live/playlist.m3u8
  -> client hls.js
  -> video 태그 재생
```

프론트가 받던 API 응답은 다음과 같았다.

```json
[
  {
    "channelId": "ch101",
    "live": {
      "playlistUrl": "/streams/ch101/live/playlist.m3u8",
      "status": "live"
    },
    "vod": null
  }
]
```

프론트는 `playlistUrl`을 서버 주소와 합쳐서 hls.js에 연결했다.

```ts
hls.loadSource("http://localhost:3000/streams/ch101/live/playlist.m3u8");
```

### 지연이 생기는 지점

HLS는 영상을 바로 보내지 않는다.

먼저 일정 시간만큼 영상을 모아 `.ts` segment 파일을 만든다.

```text
실시간 영상 입력
  -> 4초 분량 누적
  -> seg00001.ts 파일 생성
  -> playlist.m3u8 갱신
  -> 브라우저가 playlist 조회
  -> 브라우저가 segment 다운로드
  -> hls.js가 버퍼링 후 재생
```

이 과정에서 다음 시간이 누적된다.

- segment가 완성될 때까지 대기하는 시간
- playlist가 갱신될 때까지 대기하는 시간
- 브라우저가 segment를 다운로드하는 시간
- hls.js가 끊김 방지를 위해 버퍼를 확보하는 시간
- live edge보다 몇 개 segment 뒤에서 재생하는 시간

즉 HLS는 안정적인 재생에는 좋지만, 관제처럼 현재 화면을 빠르게 봐야 하는 서비스에는 지연이 크다.

---

## 3. 변경 후 구조: mediaMTX + WebRTC

변경 후에는 NestJS가 직접 FFmpeg를 실행하지 않는다.

RTSP 수신과 WebRTC/HLS 스트리밍은 mediaMTX가 담당한다.

NestJS는 다음 역할만 맡는다.

- RTSP 카메라 등록 요청 처리
- mediaMTX API 호출
- 프론트에 `hlsUrl`, `webRtcUrl` 반환
- 추후 인증, 권한, 카메라 접근 제어 담당

전체 흐름은 다음과 같다.

```text
1. mmp-server 시작
2. RTSP_CHANNELS 환경변수 읽기
3. NestJS가 mediaMTX API로 path 등록
4. mediaMTX가 RTSP 카메라를 on-demand로 연결
5. GET /streams가 hlsUrl, webRtcUrl 반환
6. 프론트가 webRtcUrl로 mediaMTX에 직접 연결
7. 브라우저 video 태그에서 WebRTC stream 재생
```

변경 후 API 응답은 다음과 같다.

```json
[
  {
    "channelId": "ch101",
    "live": {
      "hlsUrl": "http://localhost:8888/ch101/index.m3u8",
      "webRtcUrl": "http://localhost:8889/ch101/whep"
    },
    "vod": null
  }
]
```

프론트는 `webRtcUrl`이 있으면 WebRTC를 우선 사용한다.

```text
webRtcUrl 있음
  -> WebRTC Player 사용

webRtcUrl 없음, hlsUrl 있음
  -> HLS Player fallback
```

---

## 4. 한눈에 보는 흐름 비교

| 구분 | 이전 HLS 방식 | 변경 WebRTC 방식 |
| --- | --- | --- |
| RTSP 수신 | NestJS 내부 FFmpeg | mediaMTX |
| 영상 변환 | FFmpeg가 HLS 파일 생성 | mediaMTX가 WebRTC로 중계 |
| 프론트 재생 URL | `playlistUrl` | `webRtcUrl` 우선, `hlsUrl` fallback |
| 재생 방식 | hls.js + `.m3u8` | RTCPeerConnection + MediaStream |
| 영상 단위 | `.ts` 파일 segment | 실시간 media packet |
| 지연 원인 | segment 생성, playlist 갱신, 버퍼링 | ICE 연결 후 바로 media stream 수신 |
| 적합한 기능 | 녹화, 히스토리, 일반 모니터링 | 실시간 관제, PTZ, 카메라 집중 모드 |

---

## 5. WebRTC에서 지연이 줄어드는 이유

WebRTC는 HLS와 다르게 파일 기반 재생이 아니다.

HLS에서는 다음 단계가 필요했다.

```text
segment 파일 생성
playlist.m3u8 갱신
segment 다운로드
버퍼 확보
재생 시작
```

WebRTC에서는 이 단계가 사라진다.

```text
SDP offer 생성
WHEP endpoint로 offer 전송
mediaMTX가 SDP answer 반환
ICE 연결 성립
media track 수신
video.srcObject로 바로 출력
```

즉 WebRTC는 실시간 통신을 전제로 하기 때문에, 영상을 몇 초 단위 파일로 쌓아두지 않는다.

프론트에서는 remote stream을 video 태그에 직접 연결한다.

```ts
video.srcObject = remoteStream;
```

이 구조 때문에 실제 카메라 화면과 브라우저 화면 사이의 지연이 HLS보다 훨씬 작아진다.

---

## 6. Docker와 포트 구조

mediaMTX는 여러 프로토콜을 동시에 열어 둔다.

현재 로그 기준 포트는 다음과 같다.

```text
8554  -> RTSP
8888  -> HLS
8889  -> WebRTC HTTP / WHEP
8189  -> WebRTC ICE UDP
9997  -> mediaMTX API
```

`docker-compose.yml`에서는 WebRTC를 위해 `8889/tcp`뿐 아니라 `8189/udp`도 열어야 한다.

```yaml
ports:
  - "8554:8554"
  - "8888:8888"
  - "8889:8889"
  - "8189:8189/udp"
  - "9997:9997"
```

`8889`는 WebRTC 연결 협상 요청이 들어오는 HTTP 포트이다.

`8189/udp`는 실제 WebRTC ICE 연결에 사용된다.

만약 `8189/udp`가 열려 있지 않으면 mediaMTX 로그에 다음과 같은 메시지가 나올 수 있다.

```text
WebRTC session created
WebRTC session closed: deadline exceeded while waiting connection
```

이 메시지는 프론트가 WebRTC 연결 요청은 보냈지만, ICE 연결이 끝까지 성립되지 못해 timeout이 발생했다는 의미이다.

---

## 7. 현재 프론트 연결 구조

프론트는 먼저 NestJS의 `/streams` API를 호출한다.

```text
GET http://localhost:3000/streams
```

NestJS는 mediaMTX에 등록된 채널 정보를 기반으로 다음 URL을 내려준다.

```json
{
  "hlsUrl": "http://localhost:8888/ch101/index.m3u8",
  "webRtcUrl": "http://localhost:8889/ch101/whep"
}
```

프론트의 source 선택 기준은 다음과 같다.

```text
1. webRtcUrl이 있으면 WebRTC 사용
2. webRtcUrl이 없고 hlsUrl이 있으면 HLS 사용
3. 둘 다 없으면 mock camera 상태 유지
```

즉 최신 구조에서는 프론트가 `mmp-server`에서 영상 파일을 직접 받지 않는다.

프론트는 NestJS에서 URL 목록만 받고, 실제 영상은 mediaMTX로 직접 연결한다.

```text
프론트
  -> NestJS /streams로 URL 조회
  -> mediaMTX webRtcUrl로 직접 영상 수신
```

---

## 8. 이전과 변경 후 역할 분리

### 이전

```text
NestJS
  -> RTSP 연결
  -> FFmpeg 실행
  -> HLS 파일 생성
  -> playlist/segment 서빙
  -> API 응답
```

NestJS가 너무 많은 역할을 담당했다.

영상 프로세스 관리, 파일 생성, 스트림 상태 관리까지 모두 애플리케이션 서버 안에 있었다.

### 변경 후

```text
mediaMTX
  -> RTSP 연결
  -> HLS 제공
  -> WebRTC 제공
  -> 스트리밍 프로토콜 처리

NestJS
  -> 카메라 등록 API
  -> mediaMTX API 호출
  -> URL 반환
  -> 인증/권한 관리

Client
  -> /streams 조회
  -> webRtcUrl로 재생
  -> 실패 시 hlsUrl fallback 가능
```

스트리밍은 mediaMTX가 담당하고, NestJS는 비즈니스 API와 권한 제어에 집중할 수 있다.

---

## 9. 최종 정리

지연이 줄어드는 이유는 다음과 같다.

- HLS처럼 `.ts` segment를 생성하지 않는다.
- `playlist.m3u8` 갱신을 기다리지 않는다.
- 브라우저가 여러 segment를 버퍼링한 뒤 재생하지 않는다.
- WebRTC는 실시간 media packet을 직접 주고받는다.
- mediaMTX가 RTSP 입력과 WebRTC 출력을 전문적으로 처리한다.

이전과 달라진 핵심은 다음과 같다.

```text
이전
NestJS가 RTSP를 HLS 파일로 변환하고 프론트가 playlist를 재생

변경 후
mediaMTX가 RTSP를 WebRTC로 중계하고 프론트가 WebRTC로 직접 수신
```

따라서 MMP 관제 시스템에서는 다음 구조가 적합하다.

```text
실시간 관제 / PTZ / 카메라 집중 모드
  -> WebRTC

히스토리 / 녹화 / 다시보기
  -> HLS 또는 VOD
```

즉 WebRTC 전환은 단순히 재생 라이브러리를 바꾼 것이 아니라, **영상 전달 방식을 파일 기반 스트리밍에서 실시간 연결 기반 스트리밍으로 바꾼 것**이다.
