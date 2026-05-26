# RTSP to HLS 연결 구현 및 오류 정리

## 1. 전체 연결 구조

이번 구현의 핵심은 **프론트엔드가 RTSP 카메라 주소를 직접 재생하지 않고**, 서버가 RTSP를 HLS로 변환한 뒤 프론트엔드가 `hls.js`로 재생하는 구조를 만드는 것이다.

```text
IP Camera
  -> RTSP
  -> mmp-server
  -> ffmpeg
  -> HLS playlist / segment
  -> client
  -> hls.js
  -> <video>
```

역할을 나누면 다음과 같다.

- `mmp-server`: RTSP 주소를 환경변수로 읽고, `ffmpeg`로 `.m3u8` playlist와 `.ts` segment를 생성한다.
- `client`: 서버의 `/streams` API를 호출해서 HLS playlist URL을 받고, `hls.js`로 `<video>`에 붙여 재생한다.
- 브라우저: `rtsp://` 주소를 직접 재생하지 않고, HTTP 기반의 HLS URL만 재생한다.

## 2. 서버 환경변수 작성

RTSP 주소에는 카메라 계정과 비밀번호가 포함될 수 있으므로 반드시 `mmp-server/.env`에만 작성해야 한다.

`client/.env`에 RTSP 주소를 넣으면 안 된다. Vite에서 `VITE_`로 시작하는 환경변수는 브라우저 번들에 포함될 수 있기 때문이다.

### mmp-server/.env 예시

```env
PORT=3000
RECORDINGS_DIR=./recordings
HLS_OUTPUT_DIR=./hls
SEGMENT_DURATION=4
PLAYLIST_WINDOW=6
FILE_STABLE_MS=500
RTSP_CHANNELS='[{"channelId":"ch101","rtspUrl":"rtsp://USER:PASSWORD@HOST:PORT/PATH"}]'
```

주의할 점은 다음과 같다.

- `PORT`는 서버 실행 포트이다.
- `RTSP_CHANNELS`는 JSON 문자열이어야 한다.
- `channelId`는 프론트엔드에서 카메라 채널을 식별하는 값이다.
- `rtspUrl`에는 실제 카메라 RTSP 주소가 들어간다.
- 실제 계정, 비밀번호, IP 주소는 문서나 프론트엔드 코드에 남기지 않는다.

## 3. 클라이언트 환경변수 작성

클라이언트는 백엔드 서버 주소만 알면 된다.

### client/.env 예시

```env
VITE_MMP_SERVER_URL=http://localhost:3000
```

이 값은 프론트엔드에서 아래 API를 호출하는 기준 주소가 된다.

```text
GET http://localhost:3000/streams
```

서버를 다른 포트나 배포 주소로 실행하면 `VITE_MMP_SERVER_URL`도 그 주소에 맞춰 수정해야 한다.

## 4. 실행 순서

먼저 백엔드를 실행한다.

```bash
cd mmp-server
npm run start:dev
```

그 다음 프론트엔드를 실행한다.

```bash
cd client
npm run dev
```

정상 연결 여부는 아래 명령으로 확인할 수 있다.

```bash
curl http://localhost:3000/streams
```

정상이라면 아래처럼 `playlistUrl`이 포함된 응답이 나온다.

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

playlist도 직접 확인할 수 있다.

```bash
curl http://localhost:3000/streams/ch101/live/playlist.m3u8
```

정상이라면 아래처럼 `.ts` segment 목록이 나온다.

```m3u8
#EXTM3U
#EXT-X-VERSION:3
#EXT-X-TARGETDURATION:5
#EXT-X-MEDIA-SEQUENCE:30
#EXTINF:3.333000,
seg00030.ts
```

## 5. 서버에서 하는 일

`mmp-server`는 `RTSP_CHANNELS` 환경변수를 읽어서 채널을 등록한다.

서버 내부 흐름은 다음과 같다.

```text
RTSP_CHANNELS 읽기
  -> channelId, rtspUrl 파싱
  -> ffmpeg 실행
  -> hls/{channelId}/playlist.m3u8 생성
  -> hls/{channelId}/seg00000.ts 생성
  -> /streams API로 playlistUrl 제공
```

프론트엔드는 서버 파일 경로를 직접 보지 않고, HTTP API만 사용한다.

```text
/streams
/streams/:channelId/live/playlist.m3u8
/streams/:channelId/live/:segment
```

## 6. 클라이언트에서 구현한 내용

클라이언트에서는 크게 세 부분을 구현했다.

### 1. 서버 스트림 API 호출

`/streams` API를 호출해서 서버가 제공하는 채널 목록을 가져온다.

```ts
fetch(`${VITE_MMP_SERVER_URL}/streams`)
```

서버 응답의 `playlistUrl`은 상대 경로이므로, 프론트엔드에서 서버 base URL과 합쳐서 절대 URL로 만든다.

```text
/streams/ch101/live/playlist.m3u8
-> http://localhost:3000/streams/ch101/live/playlist.m3u8
```

### 2. HLS 플레이어 구현

브라우저가 HLS를 직접 지원하면 `<video>`에 바로 `src`를 넣고, 그렇지 않으면 `hls.js`를 사용한다.

기본 흐름은 다음과 같다.

```text
video ref 준비
  -> Hls.isSupported() 확인
  -> hls.loadSource(src)
  -> hls.attachMedia(video)
  -> MANIFEST_PARSED 이후 play()
  -> unmount 시 hls.destroy()
```

### 3. 카메라 월과 연결

대시보드의 카메라 월에서는 서버에서 받은 live 채널을 기존 mock 카메라 데이터 위에 덮어씌운다.

- 서버 스트림이 있으면 `protocol`을 `HLS`로 표시한다.
- `streamUrl`이 있는 카메라는 `HlsPlayer`로 실제 영상을 재생한다.
- 서버가 꺼져 있거나 스트림이 없으면 기존 mock 화면을 유지한다.

## 7. 연결 과정에서 발생한 오류와 원인

### 1. `/streams`가 빈 배열을 반환한 문제

증상은 다음과 같았다.

```json
[]
```

원인은 `mmp-server/.env`의 환경변수 키 이름이 잘못되어 서버가 RTSP 설정을 읽지 못한 것이었다.

잘못된 예시는 다음과 같다.

```env
RTSP_npm run starCHANNELS=...
```

서버 코드는 정확히 `RTSP_CHANNELS`만 읽는다.

```ts
config.get<string>('RTSP_CHANNELS', '')
```

따라서 아래처럼 수정해야 한다.

```env
RTSP_CHANNELS='[{"channelId":"ch101","rtspUrl":"rtsp://USER:PASSWORD@HOST:PORT/PATH"}]'
```

수정 후 서버를 재시작해야 한다.

### 2. `spawn ffmpeg ENOENT` 오류

증상은 다음과 같았다.

```text
Error: spawn ffmpeg ENOENT
```

이 오류는 서버가 `ffmpeg` 실행 파일을 찾지 못했다는 뜻이다.

서버는 RTSP를 HLS로 변환하기 위해 내부에서 `ffmpeg`를 실행한다.

```text
RTSP input
  -> ffmpeg
  -> HLS output
```

Mac에서는 Homebrew로 설치할 수 있다.

```bash
brew install ffmpeg
```

설치 확인은 다음 명령으로 한다.

```bash
ffmpeg -version
```

설치 후 `mmp-server`를 다시 실행해야 한다.

### 3. client 실행 시 Node 버전 오류

증상은 다음과 같았다.

```text
You are using Node.js 20.12.2. Vite requires Node.js version 20.19+ or 22.12+.
```

원인은 현재 client의 Vite 버전이 더 높은 Node 버전을 요구하기 때문이다.

해결 방법은 Node 버전을 올리는 것이다.

```bash
nvm install 22
nvm use 22
```

또는 최소 요구 버전인 Node `20.19+`를 사용해야 한다.

### 4. `HLS 연결 실패`가 표시된 문제

처음에는 서버나 RTSP 변환 문제처럼 보였지만, 실제 확인 결과 서버는 정상적으로 playlist와 segment를 제공하고 있었다.

확인한 내용은 다음과 같다.

- `/streams` 응답 정상
- `playlist.m3u8` 응답 정상
- `.ts` segment 응답 정상
- segment 코덱은 H.264로 확인됨

따라서 원인은 프론트엔드 상태 관리 쪽이었다.

이전에 실패했던 재생 상태가 열린 브라우저 화면에 남아 있거나, HLS 플레이어가 재생 중에 불필요하게 다시 생성될 수 있었다.

이를 보완하기 위해 `streamUrl` 기준으로 재생 상태를 관리하고, 재연결 시 HLS 플레이어가 명확히 다시 mount되도록 처리했다.

### 5. 계속 `HLS 연결 중...`이 반복된 문제

이 문제의 원인은 `HlsPlayer`가 계속 재생성되는 구조였다.

문제가 된 흐름은 다음과 같다.

```text
HlsPlayer mount
  -> onStatusChange('loading')
  -> CameraFeed state update
  -> CameraFeed rerender
  -> onStatusChange 함수가 새로 생성됨
  -> HlsPlayer useEffect dependency 변경
  -> 기존 HLS destroy
  -> 새 HLS 생성
  -> 다시 loading
```

즉 서버 스트림은 정상인데, 프론트엔드에서 HLS 인스턴스를 계속 없애고 다시 만들고 있었다.

해결 방법은 `CameraFeed`에서 `onStatusChange`로 넘기는 함수를 `useCallback`으로 고정하는 것이다.

```ts
const handlePlaybackStatusChange = useCallback((nextStatus) => {
  setPlayback((current) => ({
    streamUrl,
    status: nextStatus,
    retryKey: current.streamUrl === streamUrl ? current.retryKey : 0,
  }));
}, [streamUrl]);
```

이렇게 하면 `streamUrl`이 바뀌지 않는 한 `HlsPlayer`의 effect가 불필요하게 재실행되지 않는다.

## 8. 디버깅 체크리스트

영상이 안 나올 때는 아래 순서로 확인한다.

### 1. 서버가 실행 중인지 확인

```bash
lsof -iTCP:3000 -sTCP:LISTEN -n -P
```

### 2. 프론트엔드가 실행 중인지 확인

```bash
lsof -iTCP:5173 -sTCP:LISTEN -n -P
```

### 3. 서버가 스트림을 알고 있는지 확인

```bash
curl http://localhost:3000/streams
```

빈 배열이면 서버가 RTSP 설정을 읽지 못한 것이다.

### 4. playlist가 만들어졌는지 확인

```bash
curl http://localhost:3000/streams/ch101/live/playlist.m3u8
```

`404 Playlist not ready yet`가 나오면 아직 첫 segment가 생성되지 않았거나 ffmpeg가 실패한 것이다.

### 5. segment 파일이 생성되는지 확인

```bash
find mmp-server/hls/ch101 -name 'seg*.ts' | tail
```

segment가 계속 증가해야 live 변환이 진행 중인 것이다.

### 6. ffmpeg 설치 여부 확인

```bash
which ffmpeg
ffmpeg -version
```

### 7. client 환경변수 확인

```bash
cat client/.env
```

아래 값이 서버 주소와 맞아야 한다.

```env
VITE_MMP_SERVER_URL=http://localhost:3000
```

## 9. 이번 구현에서 얻은 결론

이번 연결 과정에서 가장 중요한 점은 **문제를 서버, 스트림, 브라우저, React 생명주기로 나눠서 봐야 한다는 것**이었다.

서버의 `/streams`, playlist, segment가 정상이어도 프론트엔드에서 HLS 인스턴스를 반복해서 destroy하면 영상은 계속 연결 중으로 보일 수 있다.

따라서 HLS 구현에서는 다음을 반드시 지켜야 한다.

- RTSP 주소는 서버에서만 관리한다.
- client는 서버 base URL만 환경변수로 가진다.
- ffmpeg 설치와 실행 가능 여부를 먼저 확인한다.
- `/streams` 응답이 비어 있으면 서버 env 설정부터 확인한다.
- HLS 플레이어의 `useEffect` dependency를 안정적으로 관리한다.
- 재생 상태 변경 콜백은 불필요하게 새로 생성되지 않도록 한다.
- 컴포넌트 unmount 시 `hls.destroy()`로 정리한다.

현재 프로젝트에서는 `RTSP -> mmp-server -> HLS -> hls.js -> video` 구조로 영상을 연결하는 방향이 적절하며, 이후에는 여러 카메라를 동시에 재생할 때 성능 최적화와 재연결 정책을 더 구체화해야 한다.
