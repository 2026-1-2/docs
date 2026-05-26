# HLS 지연 원인과 WebRTC 전환 정리

## 1. 먼저 결론부터

현재 HLS 방식에서 약 30초 정도 지연이 발생하는 이유는 단순히 프론트엔드 코드 문제가 아니라 **HLS 자체가 segment 기반으로 동작하는 구조**이기 때문이다.

HLS는 실시간 영상을 바로 전송하는 것이 아니라, 서버가 영상을 일정 길이의 파일로 잘라서 `.ts` segment를 만들고, 이 목록을 `.m3u8` playlist로 제공한다.

```text
RTSP Camera
  -> ffmpeg
  -> 4초 단위 .ts segment 생성
  -> playlist.m3u8 갱신
  -> hls.js가 playlist 조회
  -> segment 다운로드
  -> video 태그에서 재생
```

이 과정에서 segment 생성 시간, playlist 갱신 시간, 브라우저 버퍼링 시간이 누적되면서 실제 카메라 화면보다 늦게 보이게 된다.

반면 WebRTC는 segment 파일을 만들지 않고, 브라우저와 서버가 실시간 연결을 맺은 뒤 미디어 패킷을 바로 주고받는다.

```text
RTSP Camera
  -> WebRTC Media Server
  -> PeerConnection
  -> video.srcObject
```

따라서 관제 서비스처럼 실시간성이 중요한 화면에서는 HLS보다 WebRTC가 더 적합하다.

---

## 2. 현재 HLS 구조에서 지연이 발생하는 이유

현재 프로젝트의 HLS 연결 구조는 다음과 같다.

```text
카메라 RTSP 주소
  -> mmp-server
  -> ffmpeg
  -> hls/{channelId}/playlist.m3u8
  -> hls/{channelId}/seg00000.ts
  -> client
  -> hls.js
  -> <video>
```

이 구조에서 지연이 생기는 핵심 원인은 다음과 같다.

### 1. segment가 만들어질 때까지 기다려야 한다

현재 서버는 ffmpeg로 HLS segment를 생성한다.

예시 설정은 다음과 같다.

```env
SEGMENT_DURATION=4
PLAYLIST_WINDOW=6
```

`SEGMENT_DURATION=4`는 영상을 약 4초 단위의 파일로 자른다는 의미이다.

즉 브라우저가 영상을 받기 위해서는 최소한 서버가 첫 번째 segment 파일을 생성할 때까지 기다려야 한다.

```text
실시간 영상 입력
  -> 4초 분량 누적
  -> seg00001.ts 생성
  -> playlist.m3u8 갱신
  -> 브라우저가 다운로드
  -> 재생
```

이 단계만으로도 몇 초의 지연이 발생한다.

### 2. playlist window 때문에 최신 시점보다 뒤에서 재생한다

HLS는 보통 안정적인 재생을 위해 최신 segment 바로 앞이 아니라, 몇 개의 segment를 뒤에 두고 재생한다.

현재 설정에서 `PLAYLIST_WINDOW=6`이면 playlist에는 최대 6개의 segment가 유지된다.

```text
seg00010.ts
seg00011.ts
seg00012.ts
seg00013.ts
seg00014.ts
seg00015.ts
```

각 segment가 4초라면 playlist 전체 window는 약 24초이다.

```text
4초 * 6개 = 24초
```

hls.js도 라이브 스트림을 안정적으로 재생하기 위해 live edge에서 일정 segment만큼 떨어진 위치를 선택한다.

현재 프론트 코드에서는 다음과 같은 설정을 사용한다.

```ts
new Hls({
  enableWorker: true,
  liveSyncDurationCount: 3,
  lowLatencyMode: true,
});
```

`liveSyncDurationCount=3`이면 대략 3개 segment 정도를 기준으로 live edge보다 뒤에서 재생하려고 한다.

```text
4초 * 3개 = 약 12초
```

여기에 segment 생성, playlist 갱신, 네트워크 다운로드, 브라우저 버퍼링까지 더해지면 20~30초 수준의 지연이 발생할 수 있다.

### 3. lowLatencyMode만으로 지연이 크게 줄어들지 않는다

hls.js의 `lowLatencyMode`를 켜도 서버가 일반 HLS 방식으로 `.ts` segment만 생성한다면 지연을 크게 줄이기 어렵다.

Low Latency HLS가 제대로 동작하려면 서버도 partial segment, preload hint 같은 LL-HLS 구조를 지원해야 한다.

현재 구조는 일반적인 HLS이다.

```text
일반 HLS
  -> segment 파일이 완성된 뒤 제공

Low Latency HLS
  -> segment가 완성되기 전 일부 조각을 먼저 제공
```

따라서 프론트에서 `lowLatencyMode` 옵션을 켜는 것만으로는 30초 지연 문제가 근본적으로 해결되지 않는다.

### 4. HLS는 안정성을 위해 버퍼를 확보한다

HLS는 네트워크가 조금 불안정해도 영상이 끊기지 않도록 충분한 버퍼를 확보하려는 성격이 강하다.

이 특성은 일반 영상 시청 서비스에는 장점이다.

하지만 관제 서비스에서는 단점이 될 수 있다.

```text
영상 시청 서비스
  -> 몇 초 늦어도 끊기지 않는 것이 중요

관제 서비스
  -> 약간 불안정하더라도 현재 상황을 빨리 보는 것이 중요
```

현재 MMP 관제 시스템은 실시간 카메라 모니터링과 PTZ 제어 가능성을 고려해야 하므로, 30초 지연은 요구사항에 맞지 않을 가능성이 높다.

---

## 3. WebRTC를 사용하면 지연을 줄일 수 있는 이유

WebRTC는 HLS처럼 segment 파일을 만들지 않는다.

브라우저와 서버가 직접 미디어 연결을 맺고, 영상 데이터를 실시간 패킷 단위로 전달한다.

```text
HLS
  -> 파일 생성
  -> playlist 갱신
  -> 파일 다운로드
  -> 버퍼링 후 재생

WebRTC
  -> PeerConnection 연결
  -> 미디어 패킷 수신
  -> 바로 video 태그에 출력
```

이 차이 때문에 WebRTC는 보통 HLS보다 훨씬 낮은 지연을 기대할 수 있다.

### WebRTC 동작 흐름

WebRTC를 사용하면 전체 흐름은 다음과 같이 바뀐다.

```text
RTSP Camera
  -> mmp-server 또는 별도 Media Server
  -> RTSP를 WebRTC로 변환
  -> WebRTC signaling
  -> RTCPeerConnection
  -> MediaStream
  -> video.srcObject
```

브라우저에서는 HLS처럼 `.m3u8` 주소를 `<video>`에 넣는 것이 아니라, `RTCPeerConnection`으로 받은 `MediaStream`을 video 태그에 연결한다.

```ts
video.srcObject = remoteStream;
```

### 지연이 줄어드는 핵심 이유

WebRTC가 지연을 줄일 수 있는 이유는 다음과 같다.

- segment 파일을 생성하지 않는다.
- playlist 갱신을 기다리지 않는다.
- 실시간 전송을 전제로 설계되어 있다.
- 브라우저가 받은 media track을 바로 video 태그에 연결한다.
- 네트워크 상태에 따라 jitter buffer를 조절하며 실시간성을 우선한다.

즉 HLS에서 발생하던 아래 단계들이 사라진다.

```text
4초 segment 생성 대기
playlist.m3u8 갱신 대기
여러 segment 확보 후 재생
브라우저 HLS buffer 확보
```

그 결과 관제 화면에서는 실제 카메라 화면과의 차이를 훨씬 줄일 수 있다.

---

## 4. HLS와 WebRTC 비교

| 항목 | HLS | WebRTC |
| --- | --- | --- |
| 전송 방식 | HTTP 기반 파일 전송 | 실시간 미디어 연결 |
| 재생 단위 | `.m3u8` playlist + `.ts` segment | RTP media packet |
| 브라우저 구현 | `hls.js`로 비교적 간단 | `RTCPeerConnection`과 signaling 필요 |
| 일반 지연 | 수 초 ~ 수십 초 | 낮은 지연 |
| 안정성 | 네트워크 변동에 강함 | 네트워크와 NAT 환경에 민감 |
| 확장성 | CDN, 캐시 활용에 유리 | media server 구조 필요 |
| 관제 적합성 | 단순 모니터링에는 가능 | 실시간 관제와 제어에 적합 |
| PTZ 제어 | 화면 반응이 늦을 수 있음 | 조작 결과 확인이 빠름 |

---

## 5. 우리 프로젝트에서 WebRTC가 필요한 이유

MMP 드론 관제 시스템은 단순 영상 시청 서비스가 아니라 관제 서비스이다.

관제 서비스에서는 화면이 안정적으로 보이는 것도 중요하지만, 실제 상황과 화면 사이의 지연이 너무 커지면 문제가 된다.

예를 들어 현재처럼 약 30초 지연이 발생하면 다음 문제가 생긴다.

- 드론 이동 상황을 즉시 확인하기 어렵다.
- 위험 상황 알림과 영상 화면의 시점이 맞지 않을 수 있다.
- PTZ 카메라를 조작해도 결과가 늦게 보여 조작감이 떨어진다.
- 관제자가 현재 상황이 아니라 과거 상황을 보고 판단할 수 있다.

따라서 실시간 모니터링이 핵심인 화면은 WebRTC 전환을 고려하는 것이 맞다.

특히 아래 기능에서는 WebRTC가 더 적합하다.

- 메인 관제 화면의 live camera wall
- 카메라 집중 모드
- PTZ 직접 제어 화면
- 위험 상황 발생 시 즉시 확인해야 하는 화면

반대로 아래 기능에서는 HLS 또는 VOD 방식도 여전히 사용할 수 있다.

- 녹화 영상 조회
- 히스토리 재생
- 이벤트 발생 시점 다시보기
- 다수 사용자가 동시에 보는 비실시간 화면

---

## 6. WebRTC로 변경할 때 필요한 서버 구조

프론트엔드에서 WebRTC Player를 만드는 것만으로는 WebRTC 전환이 완성되지 않는다.

WebRTC는 HLS와 달리 서버에서 signaling과 media relay를 제공해야 한다.

필요한 서버 역할은 다음과 같다.

```text
1. RTSP 카메라 스트림 수신
2. RTSP 영상을 WebRTC로 변환 또는 relay
3. 브라우저와 SDP offer / answer 교환
4. ICE candidate 처리
5. STUN / TURN 서버 설정
6. 연결 상태 관리
```

프론트엔드가 기대하는 서버 응답 예시는 다음과 같다.

```json
[
  {
    "channelId": "cam1",
    "live": {
      "status": "live",
      "signalingUrl": "/streams/cam1/webrtc",
      "signalingProtocol": "whep",
      "iceServers": [
        {
          "urls": "stun:stun.l.google.com:19302"
        }
      ]
    },
    "vod": null
  }
]
```

프론트엔드는 `/streams`에서 `signalingUrl`을 받으면 HLS 대신 WebRTC를 우선 사용한다.

```text
signalingUrl 있음
  -> WebRTC Player 사용

playlistUrl만 있음
  -> HLS Player fallback
```

이렇게 하면 서버가 WebRTC를 지원하기 전까지는 HLS로 재생하고, WebRTC endpoint가 추가되면 자동으로 WebRTC 방식으로 전환할 수 있다.

---

## 7. 전환 시 주의할 점

WebRTC가 지연을 줄이는 데 유리하지만, HLS보다 구현 난이도는 높다.

주의할 점은 다음과 같다.

- 서버에 WebRTC media server 또는 gateway가 필요하다.
- SDP offer / answer를 처리하는 signaling API가 필요하다.
- 외부망 접속을 고려하면 TURN 서버가 필요할 수 있다.
- 여러 카메라를 동시에 WebRTC로 재생하면 서버와 클라이언트 부하가 커질 수 있다.
- H.265 카메라는 브라우저 WebRTC 호환성 문제가 있을 수 있어 H.264 인코딩을 우선 고려해야 한다.
- 관제 화면 전체를 WebRTC로 할지, 집중 모드만 WebRTC로 할지 정책이 필요하다.

따라서 바로 모든 영상을 WebRTC로 바꾸기보다 다음과 같은 단계적 전환이 현실적이다.

```text
1단계: 기존 HLS 구조 유지
2단계: WebRTC signaling endpoint 추가
3단계: 프론트에서 WebRTC 우선 재생, HLS fallback 유지
4단계: 메인 카메라 또는 집중 모드부터 WebRTC 적용
5단계: 성능 확인 후 전체 카메라 wall 적용 여부 결정
```

---

## 8. 최종 정리

현재 HLS에서 30초 정도 지연이 발생하는 이유는 다음과 같다.

- HLS가 segment 기반으로 동작한다.
- 현재 segment 길이가 약 4초 단위이다.
- playlist window와 hls.js live sync 설정 때문에 여러 segment 뒤에서 재생한다.
- segment 생성, playlist 갱신, 다운로드, 브라우저 버퍼링 시간이 누적된다.
- `lowLatencyMode`만으로는 일반 HLS 구조의 지연을 크게 줄이기 어렵다.

WebRTC를 사용하면 다음 이유로 지연을 줄일 수 있다.

- segment 파일을 만들지 않는다.
- playlist 갱신을 기다리지 않는다.
- 브라우저가 실시간 media stream을 직접 받는다.
- PTZ 제어처럼 즉각적인 화면 반응이 필요한 기능에 적합하다.

따라서 MMP 관제 서비스에서는 다음과 같이 역할을 나누는 것이 적절하다.

```text
실시간 관제 / PTZ 제어 / 카메라 집중 모드
  -> WebRTC

녹화 영상 / 히스토리 / 다시보기
  -> HLS 또는 VOD
```

즉 HLS를 완전히 버리는 것이 아니라, **실시간성이 필요한 영역은 WebRTC로 전환하고, 저장 영상이나 히스토리 조회는 HLS 또는 VOD로 유지하는 하이브리드 구조**가 가장 현실적이다.
