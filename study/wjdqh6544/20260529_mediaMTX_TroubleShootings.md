# 2026. 05. 29. (금) (Powered by Gemini 3.1)
## Question
```
> [builder 5/5] RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx .:
0.128 go: go.mod requires go >= 1.26.0 (running go 1.22.12; GOTOOLCHAIN=local)
------
Dockerfile:5
--------------------
   3 |     WORKDIR /build
   4 |     COPY . .
   5 | >>> RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx .
   6 |
   7 |     FROM alpine:3.19
--------------------
ERROR: failed to build: failed to solve: process "/bin/sh -c CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx ." did not complete successfully: exit code: 1

이건 뭐지?
```
## Answer
에러 메시지의 원인은 아주 명확합니다. **MediaMTX 소스 코드를 컴파일하려면 Go 언어 버전 1.26.0 이상이 필요한데, 현재 도커 빌더 환경에는 구버전인 1.22.12가 설치되어 있어서 발생한 버전 충돌 에러**입니다.

제가 이전에 안내해 드렸던 `Dockerfile`의 첫 줄에 `golang:1.22-alpine`으로 버전을 고정해 두었기 때문입니다. 최근 MediaMTX 코드가 업데이트되면서 더 높은 Go 버전을 요구하도록 최소 사양이 올라간 것입니다.

해결 방법은 아주 간단합니다. `Dockerfile`을 열고 맨 윗줄의 베이스 이미지(Base Image) 버전을 최신으로 올려주기만 하면 즉시 해결됩니다.

### 해결 방법 (Dockerfile 수정)

`Dockerfile`의 가장 첫 번째 줄을 다음과 같이 수정해 주세요.

**[수정 전]**

```dockerfile
FROM golang:1.22-alpine AS builder

```

**[수정 후]**

```dockerfile
# 요구사항에 맞게 1.26 버전으로 명시적으로 올리거나
FROM golang:1.26-alpine AS builder

# 또는 아예 항상 최신 버전의 Go를 가져오도록 태그를 생략 (가장 추천)
FROM golang:alpine AS builder

```

`golang:alpine`으로 변경하신 뒤 저장하고 다시 `docker build -t custom-mediamtx .` 명령어를 실행하시면, 최신 Go 컴파일러를 다운로드하여 정상적으로 빌드를 완료할 것입니다.

------------------
## Question
```
 > [builder 5/5] RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx .:
0.142 go: downloading github.com/bluenviron/gortsplib/v5 v5.5.3
0.142 go: downloading github.com/alecthomas/kong v1.15.0
0.144 go: downloading github.com/gin-gonic/gin v1.12.0
0.148 go: downloading github.com/kballard/go-shellquote v0.0.0-20180428030007-95032a82bc51
0.150 go: downloading github.com/bluenviron/mediacommon/v2 v2.8.4-0.20260520210731-a2f6c851f7b5
0.151 go: downloading github.com/google/uuid v1.6.0
0.156 go: downloading github.com/fsnotify/fsnotify v1.10.1
0.158 go: downloading github.com/MicahParks/keyfunc/v3 v3.8.0
0.167 go: downloading github.com/golang-jwt/jwt/v5 v5.3.1
0.178 go: downloading code.cloudfoundry.org/bytefmt v0.74.0
0.297 go: downloading github.com/bluenviron/gohlslib/v2 v2.3.3-0.20260521124246-e949956f870d
0.330 go: downloading github.com/matthewhartstonge/argon2 v1.5.4
0.436 go: downloading github.com/gin-contrib/pprof v1.5.4
0.455 go: downloading github.com/bluenviron/gortmplib v0.3.3-0.20260520213319-89bf8b0c2c7f
0.458 go: downloading github.com/gookit/color v1.6.1
0.458 go: downloading golang.org/x/term v0.43.0
0.458 go: downloading github.com/abema/go-mp4 v1.6.0
0.461 go: downloading github.com/Masterminds/semver/v3 v3.5.0
0.461 go: downloading github.com/go-git/go-git/v5 v5.19.1
0.574 go: downloading github.com/minio/selfupdate v0.6.0
0.610 go: downloading github.com/datarhei/gosrt v0.11.0
0.619 go: downloading github.com/pion/rtp v1.10.2
0.694 go: downloading github.com/pion/ice/v4 v4.2.5
0.694 go: downloading github.com/pion/logging v0.2.4
0.694 go: downloading github.com/pion/sdp/v3 v3.0.18
0.694 go: downloading github.com/pion/webrtc/v4 v4.2.13
0.757 go: downloading github.com/MicahParks/jwkset v0.11.0
0.757 go: downloading golang.org/x/time v0.14.0
0.757 go: downloading golang.org/x/sys v0.45.0
0.759 go: downloading github.com/gin-contrib/sse v1.1.0
0.759 go: downloading github.com/mattn/go-isatty v0.0.20
0.759 go: downloading github.com/quic-go/quic-go v0.59.0
0.759 go: downloading golang.org/x/net v0.55.0
0.839 go: downloading golang.org/x/crypto v0.52.0
0.839 go: downloading github.com/goccy/go-yaml v1.19.2
0.989 go: downloading github.com/google/gopacket v1.1.19
1.089 go: downloading github.com/asticode/go-astits v1.15.0
1.098 go: downloading github.com/ProtonMail/go-crypto v1.1.6
1.098 go: downloading dario.cat/mergo v1.0.0
1.098 go: downloading github.com/go-git/go-billy/v5 v5.9.0
1.157 go: downloading github.com/sergi/go-diff v1.3.2-0.20230802210424-5b0b94c5c0d3
1.157 go: downloading github.com/pion/rtcp v1.2.16
1.157 go: downloading github.com/xo/terminfo v0.0.0-20220910002029-abceb7e1c41e
1.158 go: downloading github.com/gorilla/websocket v1.5.3
1.158 go: downloading github.com/pion/srtp/v3 v3.0.10
1.161 go: downloading github.com/pion/interceptor v0.1.45
1.165 go: downloading github.com/pion/transport/v4 v4.0.1
1.167 go: downloading aead.dev/minisign v0.2.0
1.196 go: downloading github.com/go-playground/validator/v10 v10.30.1
1.197 go: downloading github.com/pelletier/go-toml/v2 v2.2.4
1.197 go: downloading github.com/ugorji/go/codec v1.3.1
1.209 go: downloading go.mongodb.org/mongo-driver/v2 v2.5.0
1.209 go: downloading google.golang.org/protobuf v1.36.10
1.212 go: downloading github.com/pion/randutil v0.1.0
1.228 go: downloading github.com/quic-go/qpack v0.6.0
1.285 go: downloading github.com/pion/dtls/v3 v3.1.2
1.301 go: downloading github.com/pion/mdns/v2 v2.1.0
1.328 go: downloading github.com/pion/stun/v3 v3.1.2
1.361 go: downloading github.com/pion/turn/v5 v5.0.4
1.379 go: downloading github.com/pion/datachannel v1.6.0
1.479 go: downloading github.com/pion/sctp v1.10.0
1.480 go: downloading github.com/go-git/gcfg v1.5.1-0.20230307220236-3a3c6141e376
1.480 go: downloading github.com/pjbgf/sha1cd v0.6.0
1.480 go: downloading github.com/jbenet/go-context v0.0.0-20150711004518-d14ea06fba99
1.488 go: downloading github.com/emirpasic/gods v1.18.1
1.526 go: downloading github.com/cyphar/filepath-securejoin v0.6.1
1.550 go: downloading github.com/benburkert/openpgp v0.0.0-20160410205803-c2471f86866c
1.565 go: downloading golang.org/x/text v0.37.0
1.603 go: downloading github.com/gabriel-vasile/mimetype v1.4.12
1.603 go: downloading github.com/go-playground/universal-translator v0.18.1
1.603 go: downloading github.com/leodido/go-urn v1.4.0
1.629 go: downloading github.com/asticode/go-astikit v0.30.0
1.637 go: downloading github.com/wlynxg/anet v0.0.5
1.653 go: downloading github.com/golang/groupcache v0.0.0-20241129210726-2c02b8208cf8
1.717 go: downloading github.com/kevinburke/ssh_config v1.2.0
1.736 go: downloading github.com/skeema/knownhosts v1.3.1
1.736 go: downloading github.com/xanzy/ssh-agent v0.3.3
1.762 go: downloading gopkg.in/warnings.v0 v0.1.2
1.802 go: downloading github.com/cloudflare/circl v1.6.3
1.844 go: downloading github.com/go-playground/locales v0.14.1
1.846 go: downloading github.com/klauspost/cpuid/v2 v2.3.0
7.431 internal/servers/hls/http_server.go:27:12: pattern hls.min.js: no matching files found
7.431 internal/core/core.go:43:12: pattern VERSION: no matching files found
------
Dockerfile:5
--------------------
   3 |     WORKDIR /build
   4 |     COPY . .
   5 | >>> RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx .
   6 |
   7 |     FROM alpine:3.19
--------------------
ERROR: failed to build: failed to solve: process "/bin/sh -c CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx ." did not complete successfully: exit code: 1

이건 뭐지?
```
## Answer
Go 언어 버전 업데이트는 성공적으로 적용되어 외부 패키지들을 정상적으로 다운로드하고 있습니다! 하지만 바로 다음 단계에서 MediaMTX 프로젝트의 **변경된 빌드 파이프라인 규칙**에 부딪힌 상황입니다.

### 🔍 에러 발생 원인

에러 로그의 핵심은 다음 두 줄입니다.

* `pattern hls.min.js: no matching files found`
* `pattern VERSION: no matching files found`

최근 MediaMTX 개발진은 용량 최적화와 관리를 위해 `hls.min.js` (HLS 웹 플레이어 스크립트)나 `VERSION` (버전 정보) 같은 정적 파일들을 Github 소스코드에 그대로 커밋해두지 않도록 아키텍처를 변경했습니다. 대신, 빌드를 시작하기 직전에 스크립트를 돌려서 해당 파일들을 동적으로 다운로드 및 생성한 뒤, 바이너리에 내장(`//go:embed`)하는 방식을 사용합니다.

현재 `Dockerfile`에서 `COPY . .`로 소스 코드만 덩그러니 복사한 뒤 곧바로 `go build`를 때려버렸기 때문에, 정작 내장해야 할 파일들이 준비되지 않아 컴파일러가 에러를 뱉고 멈춘 것입니다.

---

### 🛠️ 해결 방법: `go generate` 추가

해결 방법은 아주 간단합니다. `go build`를 실행하기 전에 누락된 파일들을 생성하라고 지시하는 **`go generate ./...`** 명령어를 한 줄 추가해 주면 됩니다.

`Dockerfile`의 Builder 단계(Stage 1) 부분을 다음과 같이 수정해 주세요.

**[수정할 Dockerfile 부분]**

```dockerfile
# ... (위쪽 내용 동일) ...

WORKDIR /build
COPY . .

# [추가된 핵심 라인] 
# go build 전에 필요한 정적 파일(hls.min.js, VERSION 등)을 동적으로 다운로드 및 생성합니다.
RUN go generate ./...

# 기존 빌드 명령어
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx .

# ... (아래 Runtime 단계 동일) ...

```

### 💡 참고 사항 (폐쇄망 환경 관련)

이전에 "내부망에서 사용할 것"이라고 하셨지만, 첨부해주신 로그(`go: downloading github.com/...`)를 보면 현재 도커 이미지를 빌드하고 있는 PC 자체는 외부망(인터넷)과 통신이 가능한 상태입니다.

따라서 `RUN go generate ./...`를 추가하시면 스크립트가 알아서 깃허브에서 최신 `hls.min.js`를 다운로드하여 성공적으로 깔끔한 커스텀 바이너리를 뽑아낼 것입니다. 수정한 `Dockerfile`로 다시 `docker build`를 진행해 보시기 바랍니다.

---------------
## Question
```
  13 |     RUN chmod +x /app/mediamtx
  14 | >>> RUN chmod +x /app/setup.sh

이거 왜 안되지?

  13 |     RUN sh -c "chmod +x /app/mediamtx /app/setup.sh
이렇게 해야 하나?
```
## Answer
결론부터 말씀드리면, 아래 제시해주신 `RUN sh -c ...` 방식으로 **바꾸실 필요가 없습니다.** (게다가 적어주신 명령어는 끝에 닫는 따옴표 `"`가 빠져 있어서 문법 에러가 발생합니다.)

원래 작성하셨던 `RUN chmod +x /app/setup.sh` 문법은 도커에서 완벽하게 정상적인 표준 문법입니다.

이 명령어에서 에러가 났다면, 99.9% 확률로 "해당 경로에 그 파일이 없기 때문(No such file or directory)"입니다.

### 🔍 실패하는 가장 흔한 원인 2가지

**1. `COPY` 명령어의 순서 누락 (가장 유력)**
도커는 위에서부터 아래로 순서대로 실행됩니다. `chmod`로 권한을 부여하려면, 그 전에 호스트 PC에 있는 `setup.sh` 파일을 컨테이너 내부(`/app`)로 복사하는 과정이 반드시 **먼저** 있어야 합니다.

**[잘못된 순서 또는 누락]**

```dockerfile
WORKDIR /app
RUN chmod +x /app/setup.sh  <-- 파일이 아직 없는데 권한을 주려고 해서 에러 발생!
COPY setup.sh /app/setup.sh

```

**[올바른 순서]**

```dockerfile
WORKDIR /app
# 1. 파일을 먼저 컨테이너 안으로 복사합니다.
COPY setup.sh /app/setup.sh 
# 2. 복사된 파일에 권한을 줍니다. (한 줄로 합쳐서 쓰는 것이 깔끔합니다)
RUN chmod +x /app/mediamtx /app/setup.sh 

```

*(참고: 이전에 우리가 작성했던 스크립트 이름은 `entrypoint.sh`였는데, 혹시 이름을 `setup.sh`로 바꾸신 것이 맞는지 파일명도 다시 한번 확인해 보세요.)*

**2. 윈도우(Windows)에서 파일 생성 시 줄바꿈 문제 (CRLF)**
만약 `COPY`도 제대로 했는데 에러가 난다면, 윈도우 환경에서 `setup.sh` 파일을 만들었을 때 들어가는 보이지 않는 줄바꿈 문자(`\r\n`) 때문에 리눅스 기반 도커가 파일 이름이나 내용을 정상적으로 인식하지 못하는 경우가 있습니다.
이 경우, 사용하시는 에디터(VS Code 등) 우측 하단에서 줄바꿈 형식을 **CRLF**에서 **LF**로 변경하고 저장하신 뒤 다시 빌드해 보세요.

---

### 💡 요약

`RUN chmod +x /app/mediamtx /app/setup.sh` 처럼 그냥 띄어쓰기로 구분해서 한 줄에 적는 것이 가장 깔끔하고 좋은 방법입니다.

Dockerfile 위쪽에 `COPY setup.sh /app/setup.sh` 라인이 정확히 들어 있는지, 그리고 실제 작업 폴더에 해당 파일이 존재하는지 확인해 보시기 바랍니다.

-----------
## Question
```
전체 Dockerfile 을 첨부함.
또한 setup.sh 역시 COPY . . 실행 시점에 이미 존재하고 있음.

  1 FROM golang:1.26-alpine AS builder
  2 RUN apk add --no-cache git build-base
  3 WORKDIR /build
  4 COPY . .
  5 RUN go generate ./...
  6 RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx .
  7
  8 FROM alpine:3.19
  9 WORKDIR /app
 10 RUN apk add --no-cache tzdata ca-certificates
 11 COPY --from=builder /build/mediamtx /app/mediamtx
 12 COPY mediamtx.yml /app/mediamtx.yml
 13 RUN chmod +x /app/mediamtx
 14 RUN chmod +x /app/setup.sh
 15 EXPOSE 8554 8889 8189/tcp 8189/udp
 16 ENTRYPOINT [ "/app/setup.sh" ]
 17
```
## Answer
원인이 아주 명확하게 보입니다. 도커의 **멀티 스테이지 빌드(Multi-stage Build)** 특성 때문에 발생한 착각입니다.

1단계(`builder`)에서 `COPY . .`을 통해 `setup.sh`가 복사된 것은 맞지만, **8번 라인의 `FROM alpine:3.19`가 실행되는 순간 이전 단계의 모든 파일은 날아가고 완전히 텅 빈 새로운 도화지(2단계 컨테이너)가 시작됩니다.**

11번, 12번 라인에서 `mediamtx` 바이너리와 `mediamtx.yml` 파일은 2단계 컨테이너로 명시적으로 복사(`COPY`)해 주셨지만, **`setup.sh`는 2단계로 복사하는 명령어가 빠져있습니다.** 그래서 14번 라인에서 권한을 주려고 할 때 파일을 찾을 수 없어 에러가 난 것입니다.

### 🛠️ 수정된 Dockerfile

12번 라인 아래에 `setup.sh`를 복사하는 명령어를 추가하고, 권한 부여 명령어를 한 줄로 깔끔하게 합쳤습니다.

```dockerfile
FROM golang:1.26-alpine AS builder
RUN apk add --no-cache git build-base
WORKDIR /build
# 1단계 컨테이너(builder)로 모든 소스 복사
COPY . .
RUN go generate ./...
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o mediamtx .

FROM alpine:3.19
WORKDIR /app
RUN apk add --no-cache tzdata ca-certificates

# 1단계 컨테이너(builder)에서 완성된 바이너리 가져오기
COPY --from=builder /build/mediamtx /app/mediamtx

# 호스트 PC에서 설정 파일과 스크립트를 2단계 컨테이너(실행 환경)로 복사
COPY mediamtx.yml /app/mediamtx.yml
COPY setup.sh /app/setup.sh

# 복사된 두 파일에 실행 권한 부여
RUN chmod +x /app/mediamtx /app/setup.sh

EXPOSE 8554 8889 8189/tcp 8189/udp
ENTRYPOINT [ "/app/setup.sh" ]

```

이렇게 수정하시면 2단계 실행 환경에도 스크립트가 정상적으로 안착하여 에러 없이 빌드가 완료될 것입니다.

------
## Question
```
ERR: [40:1] mapping key "logLevel" already defined at [6:1]
  37 | # https://github.com/bluenviron/mediamtx/blob/main/mediamtx.yml
  38 |
  39 |
> 40 | logLevel: info
       ^
  41 | logDestinations: [stdout, file]
  42 | logStructured: false
  43 | logFile: /app/logs/mediamtx.log
  44 |
  45 |

mediaMTX 를 실행하면 이게 왜 뜨지?
컨테이너 생성 후 mediamtx.yml 내용을 확인해 보면, 카메라에 대한 정보를 추가할때마다 기존의 데이터가 그대로 다시 입력되는 문제가 있음.

어떻게 해결을 하지?

1 #!/bin/sh
  2
  3 CONF_DIR="/app/cam_conf"
  4 YML_ORIG_FILE="/app/mediamtx-orig.yml"
  5 YML_FILE="/app/mediamtx.yml"
  6
  7 cat $YML_ORIG_FILE >> $YML_FILE
  8
  9 if [ -d "$CONF_DIR" ] && ls "$CONF_DIR"/*.conf 1> /dev/null 2>&1; then
 10     for conf_file in "$CONF_DIR"/*.conf; do
 11         in_camera_block=0
 12         while IFS= read -r line; do
 13             line=$(echo "$line" | awk '{$1=$1};1')
 14             if [ -z "$line" ] || echo "$line" | grep -q "^#"; then
 15                 continue
 16             fi
 17
 18             if [ "$line" = "camera {" ]; then
 19                 in_camera_block=1
 20                 cam_name=""
 21                 cam_ip=""
 22                 cam_port=""
 23                 cam_rtsp_path=""
 24                 username=""
 25                 password=""
 26                 continue
 27             fi
 28
 29             if [ "$in_camera_block" -eq 1 ]; then
 30                 if [ "$line" = "}" ]; then
 31                     in_camera_block=0
 32                     url="rtsp://"
 33
 34                     if [ -n "$username" ] && [ -n "$password" ]; then
 35                         url="${url}${username}:${password}@"
 36                     fi
 37
 38                     clean_path=$(echo "$cam_rtsp_path" | sed 's|^/||')
 39
 40                     url="${url}${cam_ip}:${cam_port}/${clean_path}"
 41
 42                     cat <<EOF >> "$YML_FILE"
 43
 44   $cam_name:
 45     source: $url
 46     rtspTransport: tcp
 47     sourceOnDemand: no
 48 EOF
 49                     continue
 50                 fi
 51
 52                 key=$(echo "$line" | awk '{print $1}')
 53                 val=$(echo "$line" | sed "s/^$key[ \t]*//" | sed 's/;$//')
 54                 case "$key" in
 55                     cam_name)      cam_name="$val" ;;
 56                     cam_ip)        cam_ip="$val" ;;
 57                     cam_port)      cam_port="$val" ;;
 58                     cam_rtsp_path) cam_rtsp_path="$val" ;;
 59                     username)      username="$val" ;;
 60                     password)      password="$val" ;;
 61                 esac
 62             fi
 63         done < "$conf_file"
 64     done
 65 fi
 66
 67 cat $YML_FILE
 68 exec /app/mediamtx "$YML_FILE"
```
## Answer
이 에러(`mapping key "logLevel" already defined`)와 데이터가 계속 중복해서 쌓이는 현상의 원인은 **쉘 스크립트 7번째 줄의 파일 리다이렉션 연산자(`>>`)** 때문입니다.

### 🔍 원인 분석

문제의 핵심은 아래 라인에 있습니다.

```bash
7 | cat $YML_ORIG_FILE >> $YML_FILE

```

* `>>` 기호는 기존 파일의 **맨 끝에 내용을 이어 붙이는(Append)** 역할을 합니다.
* 컨테이너가 처음 실행될 때는 문제가 없지만, 컨테이너가 재시작되거나 스크립트가 다시 실행될 때마다 기존 `mediamtx.yml` 파일 끝에 원본 설정(`mediamtx-orig.yml`)을 또다시 이어 붙이게 됩니다.
* 결과적으로 하나의 파일 안에 `logLevel: info` 같은 글로벌 설정이 두 번, 세 번 중복해서 들어가게 되고, MediaMTX의 YAML 파서가 "이미 정의된 키(logLevel)가 또 나왔다"며 에러를 뿜고 죽어버리는 것입니다.

### 🛠️ 해결 방법

컨테이너가 켜질 때마다 무조건 **백지상태(원본 파일)로 초기화(Overwrite)한 뒤** 카메라 정보를 추가하도록 연산자를 `>`로 바꾸거나 `cp` 명령어를 사용해야 합니다.

**[수정 전]**

```bash
cat $YML_ORIG_FILE >> $YML_FILE

```

**[수정 후] (둘 중 하나 사용)**

```bash
# 방법 1: 덮어쓰기 연산자(>) 사용
cat "$YML_ORIG_FILE" > "$YML_FILE"

# 방법 2: 복사 명령어(cp) 사용 (가장 직관적이고 추천함)
cp "$YML_ORIG_FILE" "$YML_FILE"

```

---

### 💡 수정된 전체 스크립트

7번째 줄을 `cp` 명령어로 깔끔하게 수정한 전체 스크립트입니다. 이것을 그대로 덮어쓰시면 재시작을 몇 번을 하더라도 설정이 꼬이거나 중복되는 일이 발생하지 않습니다.

```bash
#!/bin/sh

CONF_DIR="/app/cam_conf"
YML_ORIG_FILE="/app/mediamtx-orig.yml"
YML_FILE="/app/mediamtx.yml"

# [핵심 수정 부분] '>>' 가 아닌 cp를 사용하여 매번 원본 상태로 덮어쓰기(초기화) 합니다.
cp "$YML_ORIG_FILE" "$YML_FILE"

if [ -d "$CONF_DIR" ] && ls "$CONF_DIR"/*.conf 1> /dev/null 2>&1; then
    for conf_file in "$CONF_DIR"/*.conf; do
        in_camera_block=0
        while IFS= read -r line; do
            line=$(echo "$line" | awk '{$1=$1};1')
            if [ -z "$line" ] || echo "$line" | grep -q "^#"; then
                continue
            fi

            if [ "$line" = "camera {" ]; then
                in_camera_block=1
                cam_name=""
                cam_ip=""
                cam_port=""
                cam_rtsp_path=""
                username=""
                password=""
                continue
            fi

            if [ "$in_camera_block" -eq 1 ]; then
                if [ "$line" = "}" ]; then
                    in_camera_block=0
                    url="rtsp://"

                    if [ -n "$username" ] && [ -n "$password" ]; then
                        url="${url}${username}:${password}@"
                    fi

                    clean_path=$(echo "$cam_rtsp_path" | sed 's|^/||')

                    url="${url}${cam_ip}:${cam_port}/${clean_path}"

                    # 여기는 카메라 정보를 이어 붙여야 하므로 '>>' 를 쓰는 것이 맞습니다.
                    cat <<EOF >> "$YML_FILE"
  $cam_name:
    source: $url
    rtspTransport: tcp
    sourceOnDemand: no
EOF
                    continue
                fi

                key=$(echo "$line" | awk '{print $1}')
                val=$(echo "$line" | sed "s/^$key[ \t]*//" | sed 's/;$//')
                case "$key" in
                    cam_name)      cam_name="$val" ;;
                    cam_ip)        cam_ip="$val" ;;
                    cam_port)      cam_port="$val" ;;
                    cam_rtsp_path) cam_rtsp_path="$val" ;;
                    username)      username="$val" ;;
                    password)      password="$val" ;;
                esac
            fi
        done < "$conf_file"
    done
fi

cat "$YML_FILE"
exec /app/mediamtx "$YML_FILE"

```