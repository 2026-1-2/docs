# 2026. 05. 29. (금) (Powered by Gemini 3.1)
## Question
```
이러한 mediamtx 설정을 사용한다면, API 를 사용해서 카메라 추가가 가능한가?
(yml 별첨)
```
## Answer
**결론부터 말씀드리면, 제공해주신 설정으로는 API를 사용한 카메라(Path) 추가가 불가능합니다.**

이유는 크게 두 가지입니다.

### 1. API 비활성화 (`api: false`)

설정 파일을 보면 `api` 항목이 `false`로 지정되어 있습니다.

```yaml
api: false

```

이 설정으로 인해 MediaMTX의 REST API 서버 자체가 구동되지 않습니다. 따라서 외부에서 설정을 동적으로 변경하거나 카메라를 추가하는 API 호출(예: `POST /v3/config/paths/add/...`)을 보낼 수 없습니다.

### 2. 동적 경로 생성 차단 (`paths` 설정)

`paths:` 하위 항목의 `all_others` 설정을 보면 보안이 매우 엄격하게 걸려 있습니다.

```yaml
paths:
  all_others:
    publishUser: "!blocked_Not_Used!"
    readUser: "!blocked_Not_Used!"

```

이 설정은 `mediamtx.yml`에 사전에 명시적으로 정의되지 않은 임의의 모든 경로(새로운 카메라 등)에 대한 퍼블리싱(Publish)과 읽기(Read)를 원천적으로 차단합니다.

---

### 해결 방법

API를 통해 카메라를 동적으로 추가하고 관리하고 싶으시다면, 설정 파일을 다음과 같이 수정해야 합니다.

1. **API 기능 켜기**: `api: false`를 `api: true`로 변경합니다. (기본적으로 `localhost:9997` 포트로 API 서버가 활성화됩니다.)
2. **API 포트 확인 (선택)**: 필요하다면 `apiAddress: :9997`과 같이 포트를 명시할 수 있습니다.

**참고:** API를 활성화(`api: true`)한 후에는, HTTP POST 요청을 MediaMTX API 엔드포인트로 전송하여 프로그램을 재시작하지 않고도 새로운 카메라(Path) 설정을 동적으로 밀어넣을 수 있습니다.

------------------------
## Question
```
그러면 api:true 로 바꾸기만 하면, API 기능을 문제없이 이용할 수 있는건가?
```
## Answer
결론부터 말씀드리면, `api: true`로 변경하면 API 기능 자체는 활성화되지만, **실제 운영 환경에서 에러 없이 카메라를 추가하고 관리하려면 다음 3가지 요소를 추가로 확인하셔야 합니다.**

### 1. 설정 파일(`mediamtx.yml`) 쓰기 권한 (핵심)

MediaMTX는 API를 통해 새로운 카메라(Path)를 추가하거나 기존 설정을 변경할 때, 그 변경 사항을 메모리에만 유지하는 것이 아니라 **`mediamtx.yml` 파일에 직접 덮어써서 영구 저장합니다.**

* Docker 컨테이너 환경에서 구동할 경우, 볼륨 마운트 시 설정 파일을 읽기 전용(예: `-v ./mediamtx.yml:/mediamtx.yml:ro`)으로 설정했다면 API 호출 시 파일 쓰기 권한 에러(Permission Denied)가 발생합니다.
* 반드시 MediaMTX 프로세스가 해당 설정 파일에 대한 쓰기(Write) 권한을 갖도록 마운트 및 파일 권한을 수정해야 합니다.

### 2. API 포트 포워딩 및 접근 주소 (`apiAddress`)

API 서버는 기본적으로 TCP `9997` 포트를 사용합니다.

* 외부 스크립트나 호스트 머신에서 컨테이너 내부의 API를 호출하려면, 컨테이너 실행 시 반드시 **`-p 9997:9997`** 옵션을 추가해 포트를 열어주어야 합니다.
* 또한, 특정 IP에서만 접근을 허용하거나 모든 외부 접근을 허용하기 위해 설정 파일에 `apiAddress: :9997` (모든 인터페이스에서 허용) 항목을 명시적으로 추가해 두는 것이 안전하고 확실합니다.

### 3. `all_others` 차단 정책과의 관계

앞서 확인했던 `paths.all_others`의 엄격한 차단(`!blocked_Not_Used!`) 설정 자체는 API 사용에 문제가 되지 않습니다.

* API(`POST /v3/config/paths/add/{이름}`)를 통해 새로운 카메라 경로를 등록하면, 해당 카메라는 설정 파일 내에 명시적인 고유 경로로 새롭게 추가됩니다.
* 따라서 사전에 정의되지 않은 경로를 묶어서 처리하는 `all_others`의 차단 규칙을 적용받지 않으며, 새로 추가한 설정대로 정상 퍼블리싱 및 스트리밍이 가능해집니다.

---

**요약하자면:**
`api: true` 설정과 함께 **외부에서 호출 가능한 포트(9997)가 열려 있는지**, 그리고 MediaMTX가 **설정 파일에 변경 사항을 기록할 쓰기 권한이 있는지**를 세팅해 주시면, 프로그램을 재시작하지 않고도 API로 카메라를 자유롭게 추가하실 수 있습니다.
-----------------------
## Question
```
그러면 설정을 이렇게 바꾸면 되겠나?
<Mediamtx.yml>
api:true
apiAddress: :9997
<docker-compose>
ports:
- "8889:8889"
- "8189:8189/udp"
- "9997:9997" (추가)
```
## Answer
네, 방향은 아주 정확합니다! 제시해주신 대로 설정하시면 호스트 머신에서 컨테이너 내부의 MediaMTX API로 정상적인 접근이 가능해집니다.

다만, 실제 구동 시 에러가 나지 않도록 **두 가지 디테일**만 살짝 보완하시면 완벽합니다.

### 1. YAML 문법 (띄어쓰기)

YAML 파일은 콜론(`:`) 뒤에 반드시 **공백이 하나** 있어야 문법 오류 없이 정상적으로 인식됩니다.

```yaml
# mediamtx.yml 수정
api: true          # api:true 대신 공백 추가
apiAddress: :9997

```

### 2. 볼륨 마운트 쓰기 권한 확인 (가장 중요)

포트는 완벽하게 추가하셨습니다. 이제 이전 답변에서 강조했던 **설정 파일 쓰기 권한**을 확인해야 합니다. API로 카메라를 추가하면 MediaMTX가 `mediamtx.yml` 파일을 직접 수정하므로, `docker-compose.yml`의 `volumes` 설정에 읽기 전용(`:ro`) 옵션이 없어야 합니다.

전체적인 `docker-compose.yml`의 형태는 다음과 같아야 합니다.

```yaml
# docker-compose.yml 예시
services:
  mediamtx:
    image: bluenviron/mediamtx:latest
    ports:
      - "8554:8554"     # RTSP (기존에 사용 중이시라면 유지)
      - "8889:8889"     # WebRTC TCP
      - "8189:8189/udp" # WebRTC UDP
      - "9997:9997"     # (추가) API 포트
    volumes:
      # 핵심: 아래처럼 마운트되어야 하며, 끝에 ':ro' (Read-Only)가 붙어있다면 지워주세요.
      - ./mediamtx.yml:/mediamtx.yml

```

---

**💡 설정 적용 후 테스트 방법**

위와 같이 수정하신 후 컨테이너를 재시작(`docker compose up -d`)하고, 호스트 서버 터미널에서 아래 명령어를 실행해 보세요.

```bash
curl http://localhost:9997/v3/config/global/get

```

현재 MediaMTX의 전체 설정 내용이 화면에 JSON 형태로 쭉 출력된다면 API 통신 준비가 완벽하게 끝난 것입니다. 이제 POST 요청을 통해 카메라(Path)를 마음껏 추가하고 삭제하실 수 있습니다!

------------------
## Question
```
최초 컨테이너 구동 시에는 디스크의 conf 를 읽어서 카메라를 읽어오도록 하고 있음.
그러기 위해서 orig 파일을 ro 로 마운트하고, 컨테이너 시작 단계에서 mediamtx.yml 파일을 동적으로 생성해서, 생성된 yml 파일을 기준으로 mediamtx 를 시작함.

이러한 상황이라면.

- "${MEDIA_SERVER_CONF_PATH_ROOT}:/app/mediamtx-orig.yml:ro"
- "${MEDIA_SERVER_CONF_PATH_ROOT}:/app/mediamtx.yml"

이렇게 가면 되겠나? host 경로는 무시하고.
```
## Answer
**결론부터 말씀드리면, 제시해주신 마운트 방식은 논리적인 충돌과 데이터 유실을 유발할 수 있어 그대로 사용하시면 안 됩니다.**

구상하신 "최초 구동 시 원본(orig)을 읽어 동적으로 `mediamtx.yml`을 생성한다"는 아이디어는 매우 좋습니다. 하지만 도커 볼륨 설정에서 다음과 같은 두 가지 큰 문제가 발생합니다.

### 1. 발생하는 문제점

* **동일한 호스트 파일 참조 충돌**
`${MEDIA_SERVER_CONF_PATH_ROOT}` 변수가 만약 호스트(Host)의 **동일한 단일 파일**(예: `./config/mediamtx.yml`)을 가리킨다면, 컨테이너 내부의 `/app/mediamtx-orig.yml`과 `/app/mediamtx.yml`은 결국 껍데기만 다를 뿐 호스트의 같은 파일을 바라보게 됩니다.
이 상태에서 시작 스크립트가 `/app/mediamtx.yml`을 새로 생성하거나 덮어쓰면, 호스트의 원본 파일 자체가 변형되어 버리므로 `orig` 파일을 읽기 전용(`:ro`)으로 마운트한 의미가 사라집니다.
* **API로 추가한 카메라(Path) 설정 증발 (가장 치명적)**
이전 대화에서 API를 통해 카메라를 추가(`api: true`)하기로 하셨습니다. API로 카메라를 추가하면 MediaMTX는 `/app/mediamtx.yml` 파일에 그 내용을 기록합니다.
하지만 현재 구상하신 대로 '컨테이너가 시작될 때마다 orig 파일을 기준으로 yml을 새로 생성'하게 되면, 컨테이너가 재시작될 때마다 API로 열심히 추가해 두었던 카메라 설정들이 모두 초기화(삭제)되어 버립니다.

---

### 2. 올바른 해결 방안

API로 추가한 카메라 설정이 컨테이너 재시작 후에도 유지되어야 하는지(영구 저장), 아니면 휘발되어도 상관없는지에 따라 마운트 방식을 다르게 가져가야 합니다.

#### 방안 A: API로 추가한 카메라 설정을 영구 저장하고 싶은 경우 (권장)

호스트 머신에 '원본 템플릿 파일'과 '실제 구동 및 저장될 파일'을 아예 분리해야 합니다. 컨테이너 시작 스크립트는 `mediamtx.yml`이 **없을 때만** `orig`에서 복사해 오도록 로직을 짜야 합니다.

```yaml
volumes:
  # 1. 훼손되지 않는 원본 템플릿 파일 (Host의 특정 파일 -> Container의 orig)
  - "./template/mediamtx-template.yml:/app/mediamtx-orig.yml:ro"
  
  # 2. 실제로 구동되고 API 변경 사항이 저장될 파일 (Host의 빈 파일/존재하는 파일 -> Container의 yml)
  - "${MEDIA_SERVER_CONF_PATH_ROOT}:/app/mediamtx.yml"

```

*(주의: 컨테이너 시작 스크립트(Entrypoint)에서 `if [ ! -f /app/mediamtx.yml ]; then cp /app/mediamtx-orig.yml /app/mediamtx.yml; fi` 와 같이 최초 1회만 복사하도록 구성해야 API 데이터가 날아가지 않습니다.)*

#### 방안 B: 재시작 시 카메라가 다 날아가도 상관없는 경우 (완전한 휘발성)

오직 구동 시점에만 동적으로 설정을 만들고, 재시작하면 무조건 초기화되어도 괜찮다면, 실제 구동용 `mediamtx.yml`은 호스트와 마운트할 필요가 전혀 없습니다. 컨테이너 내부의 임시 파일 시스템에 생성되도록 내버려 두면 됩니다.

```yaml
volumes:
  # 1. 설정 생성을 위한 원본 파일만 읽기 전용으로 마운트
  - "${MEDIA_SERVER_CONF_PATH_ROOT}:/app/mediamtx-orig.yml:ro"
  # 2. /app/mediamtx.yml 은 볼륨 마운트에서 아예 제외합니다. (컨테이너 내부에서 스크립트가 생성)

```

---------------
## Question
```
그러면 마운트를
- "${MEDIA_SERVER_CONF_TEMPLATE_PATH}:/app/mediamtx-template.yml:ro"
      - "${MEDIA_SERVER_CONF_PROD_PATH}:/app/mediamtx.yml"

이렇게 잡고

setup.sh 스크립트 (컨테이너 시작 시 자동으로 실행되는 스크립트; 카메라 추가를 위함) 는

#!/bin/sh

CONF_DIR="/app/cam_conf"
YML_ORIG_FILE="/app/mediamtx-orig.yml"
YML_FILE="/app/mediamtx.yml"

if [ -f ${YML_FILE} ]; then
    exec /app/mediamtx "$YML_FILE"
fi

cat $YML_ORIG_FILE >> $YML_FILE

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
                    cam_name)      cam_name="$val" ;;
                    cam_ip)        cam_ip="$val" ;;
                    cam_port)      cam_port="$val" ;;
                    cam_rtsp_path) cam_rtsp_path="$val" ;;
                    username)      username="$val" ;;
                    password)      password="$val" ;;
                esac
            fi
        done < "$conf_file"
    done
fi

exec /app/mediamtx "$YML_FILE"

이렇게 가면 되나?
```
## Answer
작성하신 스크립트와 마운트 설정의 논리적인 흐름은 매우 훌륭합니다.

`if [ -f ${YML_FILE} ]; then exec ...` 구문을 통해 최초 1회만 설정 파일을 생성하고, 이후 재시작 시에는 기존 파일(API 변경 사항이 포함된 파일)을 그대로 사용하여 컨테이너를 구동하는 목적을 정확히 달성할 수 있습니다.

다만, 완벽한 동작을 위해 **세 가지 수정 및 확인이 필요**합니다.

### 1. 파일명 불일치 수정

Docker 마운트 설정과 쉘 스크립트 내의 변수명이 서로 다릅니다. 이를 일치시켜야 합니다.

* **마운트 설정:** `/app/mediamtx-template.yml`로 마운트
* **스크립트 변수:** `YML_ORIG_FILE="/app/mediamtx-orig.yml"`로 지정됨

**해결:** 스크립트의 변수를 마운트된 파일명에 맞게 수정합니다.

```bash
YML_ORIG_FILE="/app/mediamtx-template.yml"

```

### 2. 설정 파일 복사 방식 변경

빈 파일에 원본 내용을 넣을 때 `cat ... >> ...` (추가) 방식보다는 명시적으로 파일을 복사(`cp`)하는 것이 의도치 않은 쓰레기 값(Garbage)이 들어가는 것을 방지하고 더 안전합니다.

**해결:**

```bash
# 변경 전
cat $YML_ORIG_FILE >> $YML_FILE

# 변경 후
cp "$YML_ORIG_FILE" "$YML_FILE"

```

### 3. YAML 들여쓰기(Indentation) 및 `paths:` 위치 확인 (가장 중요)

스크립트 하단의 `cat <<EOF >> "$YML_FILE"` 부분은 파일의 맨 끝에 텍스트를 단순 추가합니다. MediaMTX에서 카메라 경로는 반드시 `paths:` 하위에 선언되어야 합니다.

따라서 이 스크립트가 정상 작동하려면 **두 가지 조건**이 반드시 충족되어야 합니다.

1. **템플릿 파일 구조:** 템플릿(`mediamtx-template.yml`)의 맨 마지막 내용이 반드시 `paths:` 블록이어야 합니다.
```yaml
# mediamtx-template.yml의 끝부분 예시
paths:
  all_others:
    publishUser: "!blocked_Not_Used!"
    readUser: "!blocked_Not_Used!"

```


2. **Here-Doc 들여쓰기:** 쉘 스크립트에서 추가하는 `$cam_name:` 앞에 **공백 2개**가 있어야 `paths:`의 하위 항목으로 정상 인식됩니다.

**해결 (스크립트 부분 수정):**

```bash
                    # $cam_name 앞에 띄어쓰기 2칸 필수
                    cat <<EOF >> "$YML_FILE"
  $cam_name:
    source: $url
    rtspTransport: tcp
    sourceOnDemand: no
EOF

```

---

### 최종 스크립트 정리본

위의 수정 사항을 반영한 스크립트는 다음과 같습니다.

```bash
#!/bin/sh

CONF_DIR="/app/cam_conf"
YML_ORIG_FILE="/app/mediamtx-template.yml" # 수정됨
YML_FILE="/app/mediamtx.yml"

# 이미 파일이 있으면 생성 로직 건너뛰고 바로 실행 (API 추가 사항 보존)
if [ -f "${YML_FILE}" ]; then
    exec /app/mediamtx "$YML_FILE"
fi

# 원본 템플릿 복사 (수정됨)
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

                    # 들여쓰기 2칸 유지 (중요)
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

exec /app/mediamtx "$YML_FILE"

```

이 구조대로라면 처음에 의도하신 대로 커스텀 `.conf` 포맷의 설정들을 읽어들여 안전하게 `mediamtx.yml`을 초기 구성하고, 이후 API를 통한 추가/수정 사항까지 영구적으로 유지할 수 있습니다.

----------
## Question
```
이거 파일 내용이 없을 때 cp를 수행하도록 바꿔야 할 것 같음.
```
## Answer
말씀하신 부분이 맞습니다! 도커에서 볼륨을 마운트할 때, 호스트 쪽에 파일이 없으면 도커가 빈 디렉토리나 빈 파일을 임의로 생성해 버리는 경우가 종종 발생합니다.

단순히 파일 존재 여부(`-f`)만 체크하고 빠져나가면, **내용이 0바이트인 빈 파일일 때도 스크립트가 실행을 멈추고 빈 파일로 MediaMTX를 구동하려다 에러가 날 수 있습니다.**

따라서 스크립트 구조를 "파일이 없거나 파일 크기가 0(빈 파일)일 때만 템플릿 복사 및 파싱을 수행"하는 구조로 묶어주는 것이 훨씬 안전하고 정석적입니다.

쉘 스크립트의 `-s` 옵션(파일이 존재하고 크기가 0보다 큰지 확인)을 활용하여 다음과 같이 수정하면 완벽합니다.

### 수정된 전체 setup.sh 스크립트

```bash
#!/bin/sh

CONF_DIR="/app/cam_conf"
YML_ORIG_FILE="/app/mediamtx-template.yml"
YML_FILE="/app/mediamtx.yml"

# YML_FILE이 존재하지 않거나(!), 크기가 0인 빈 파일(-s)일 경우에만 블록 내부 실행
if [ ! -s "$YML_FILE" ]; then
    
    # 1. 템플릿 파일 복사
    cp "$YML_ORIG_FILE" "$YML_FILE"

    # 2. conf 파일들을 읽어서 카메라 경로 추가
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

                        # 들여쓰기 2칸 유지 (중요)
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
fi

# 위의 조건문을 거친 후 최종적으로 mediamtx 실행 
# (기존 파일이 있으면 파싱 블록을 건너뛰고 바로 실행됨)
exec /app/mediamtx "$YML_FILE"

```

### 주요 변경점

* **`[ ! -s "$YML_FILE" ]` 조건문 사용:** 단순히 `-f`로 파일 유무만 보지 않고, 파일 내용물이 실질적으로 존재하는지(0바이트가 아닌지)까지 확인합니다.
* **`exec` 위치 이동:** 스크립트 중간에 있던 `exec` 명령을 맨 아래로 내렸습니다. 이렇게 하면 쉘 스크립트의 흐름이 한눈에 들어오고, "설정 구성 -> 서버 실행" 이라는 논리적 순서가 명확해집니다.
----------
## Question
```
이렇게 하면 되나?

if [ ! -s ${YML_FILE} ]; then
```
## Answer
네, 논리적으로 아주 정확합니다!

`! -s` 옵션을 사용하면 파일이 아예 존재하지 않거나, 존재하더라도 내용이 없는 0바이트 빈 파일일 경우 모두 참(True)으로 판별하기 때문에 의도하신 상황에 완벽하게 들어맞습니다.

한 가지 사소하지만 아주 중요한 쉘 스크립트 작성 팁을 드리자면, **변수를 호출할 때는 큰따옴표(`" "`)로 감싸주는 것이 정석이자 가장 안전한 방법입니다.**

```bash
# 추천하는 작성 방식
if [ ! -s "$YML_FILE" ]; then

```

`${YML_FILE}`처럼 괄호를 쓰는 것도 좋지만, 큰따옴표 없이 변수를 사용하면 만약 변수 값에 공백이 포함되어 있거나 변수가 비어있을 때 쉘에서 문법 에러(unexpected operator 등)를 뱉어낼 수 있습니다.

현재 스크립트에서는 경로에 공백이 없어서 큰 문제가 되진 않지만, 호환성과 안정성을 위해 큰따옴표로 묶어주시면 결점 없는 완벽한 코드가 됩니다!