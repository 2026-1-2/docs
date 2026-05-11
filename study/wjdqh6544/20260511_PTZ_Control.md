# 2026. 05. 11. (월) (Powered by Gemini 3.1)
## Question
```
파이썬으로 onvif 통해 카메라를 제어할 때

 mycam = ONVIFCamera(cam_ip, cam_port, username, password)
        media_service = mycam.create_media_service()
        ptz_service = mycam.create_ptz_service()
        profiles = media_service.GetProfiles()
        print(profiles[1])

여기서 profile 0, 1, ... 에 따른 차이가 있나?
```
## Answer
ONVIF에서 `GetProfiles()`를 통해 가져오는 프로파일들은 카메라가 제공하는 '설정 세트'라고 보시면 됩니다. 결론부터 말씀드리면, **각 프로파일 인덱스(0, 1...)에 따라 화질, 대역폭, 지원 기능이 크게 다를 가능성이 높습니다.**

일반적으로 제조사들이 프로파일을 나누는 기준은 다음과 같습니다.

### 1. 스트림 품질 (Main vs Sub Stream)

가장 흔한 차이점은 해상도와 프레임 레이트입니다.

* **Profile [0] (Main Stream):** 주로 고해상도(4K, 1080p 등)와 높은 비트레이트를 가집니다. 녹화용이나 정밀 모니터링용으로 사용됩니다.
* **Profile [1] (Sub Stream):** 모바일 기기나 다중 분할 화면용으로 낮은 해상도(VGA, 720p 등)와 낮은 비트레이트를 가집니다. 네트워크 대역폭을 아끼기 위해 사용됩니다.

### 2. 인코딩 방식 (Codec)

동일한 해상도라도 코덱 설정이 다를 수 있습니다.

* **Profile [0]:** 최신 압축 기술인 **H.265**를 사용하여 효율을 높임.
* **Profile [1]:** 호환성을 위해 **H.264**나 **MJPEG** 방식을 사용함.

### 3. PTZ 및 부가 기능 연결 여부

모든 프로파일이 PTZ(Pan-Tilt-Zoom) 기능을 지원하는 것은 아닙니다.

* 특정 프로파일에는 **PTZ Configuration**이 연결되어 있어 제어가 가능하지만, 다른 프로파일(예: 고정 화각용)에는 해당 설정이 빠져 있을 수 있습니다.
* 만약 `ptz_service.GetConfiguration(profile_token)`을 호출했을 때 설정값이 없다면, 해당 프로파일로는 PTZ 제어가 불가능합니다.

---

---

### 💡 확인해봐야 할 포인트

코드에서 `profiles[1]`을 사용하시기 전에, 실제로 어떤 설정이 들어있는지 확인해보는 것이 좋습니다.

```python
for i, profile in enumerate(profiles):
    print(f"--- Profile [{i}] ---")
    print(f"Name: {profile.Name}")
    print(f"Token: {profile.token}")
    
    # 영상 인코딩 설정 확인
    if hasattr(profile, 'VideoEncoderConfiguration'):
        print(f"Resolution: {profile.VideoEncoderConfiguration.Resolution.Width}x{profile.VideoEncoderConfiguration.Resolution.Height}")
        print(f"Encoding: {profile.VideoEncoderConfiguration.Encoding}")
    
    # PTZ 설정 포함 여부 확인
    if hasattr(profile, 'PTZConfiguration'):
        print("PTZ: Supported")
    else:
        print("PTZ: Not Supported")
    print("\n")

```

**주의사항:**
인덱스 번호(`[0]`, `[1]`)는 카메라 제조사마다 부여하는 순서가 다릅니다. 어떤 카메라는 0번이 고화질이지만, 어떤 카메라는 0번이 저화질일 수도 있습니다. 따라서 실무에서는 인덱스 번호보다는 `profile.Name`이나 `profile.token`을 체크하여 원하는 프로파일을 선택하는 것이 안전합니다.

혹시 특정 카메라 모델에서 PTZ 제어가 안 되거나 영상이 안 나오는 문제가 있어 질문하신 건가요? 상황을 더 알려주시면 구체적인 해결책을 드릴 수 있습니다.
--------
## Question
```
그럼 getProfiles() 를 했을 때 None 이 나오면, PTZ 미지원으로 보면 되나?
```
## Answer
`GetProfiles()`가 `None` 또는 빈 리스트(`[]`)를 반환한다고 해서 반드시 "PTZ 미지원"으로 단정 지을 수는 없습니다. 이는 PTZ 여부보다는 **미디어 서비스 자체의 연결이나 권한, 혹은 카메라의 설정 문제**일 가능성이 훨씬 큽니다.

ONVIF 아키텍처 상의 관계를 이해하면 원인 파악이 쉽습니다.

### 1. 서비스 간의 관계 (Media vs PTZ)

ONVIF에서 **Media Service**와 **PTZ Service**는 별개의 엔드포인트를 가집니다.

* **Media Service:** 카메라의 영상 스트림(해상도, 코덱, 프레임 등)을 관리합니다.
* **PTZ Service:** 카메라의 하드웨어 움직임(Pan, Tilt, Zoom)을 관리합니다.

`GetProfiles()`는 Media Service의 기능입니다. 만약 여기서 프로파일을 가져오지 못한다면, PTZ 제어를 하기 위한 '대상(Token)' 자체가 식별되지 않기 때문에 PTZ 명령을 내릴 수 없는 상태가 되는 것입니다.

### 2. `None`이나 빈 값이 나오는 주요 원인

PTZ 하드웨어 유무와 상관없이 다음과 같은 이유로 프로파일이 조회되지 않을 수 있습니다.

* **인증 문제 (Authentication):** ONVIF 계정 권한이 'Administrator'가 아닌 'User'나 'Operator'로 되어 있어 미디어 정보를 읽어오지 못하는 경우가 많습니다.
* **ONVIF 활성화 여부:** 일부 카메라는 웹 설정 페이지에서 'ONVIF 허용' 및 '인증 체크' 설정을 수동으로 켜줘야 합니다.
* **프로파일 생성 전:** 공장 초기화 상태의 카메라는 기본 프로파일이 생성되어 있지 않을 수 있습니다. 이 경우 제조사 앱이나 웹 페이지에서 스트림 설정을 한 번 저장해야 프로파일이 생성됩니다.
* **ONVIF 버전 불일치:** 카메라가 지원하는 ONVIF 버전과 사용 중인 라이브러리의 규격이 맞지 않을 때 발생합니다.

### 3. PTZ 지원 여부를 정확히 확인하는 방법

카메라가 PTZ를 지원하는지 기술적으로 확인하려면 아래의 단계별 체크가 필요합니다.

#### A. PTZ 서비스 객체 생성 확인

```python
try:
    ptz_service = mycam.create_ptz_service()
    print("PTZ Service 생성 성공")
except:
    print("이 카메라는 PTZ 서비스를 지원하지 않거나 엔드포인트가 없습니다.")

```

#### B. PTZ Nodes 조회

PTZ 서비스가 생성되더라도 하드웨어가 없을 수 있습니다. 이때는 `GetNodes`를 호출하여 실제 제어 가능한 '노드'가 있는지 확인해야 합니다.

```python
nodes = ptz_service.GetNodes()
if not nodes:
    print("PTZ 서비스는 응답하지만, 제어 가능한 하드웨어 노드가 없습니다. (고정형 카메라)")
else:
    print(f"PTZ 지원 확인: {len(nodes)}개의 노드 발견")

```

#### C. GetCapabilities 확인

가장 표준적인 방법은 디바이스의 기능을 직접 묻는 것입니다.

```python
capabilities = mycam.devicemgr.GetCapabilities({'Category': 'All'})
has_ptz = hasattr(capabilities, 'PTZ')
print(f"PTZ Capability: {has_ptz}")

```

### 🛠️ 결론 및 조치 사항

1. **`GetProfiles()`가 비어있다면:** 먼저 카메라 웹 설정 페이지에 접속하여 **ONVIF 계정**이 올바른지, **Media Profile**이 최소 하나 이상 생성되어 있는지 확인하세요.
2. **PTZ 여부를 알려면:** `GetProfiles()`의 결과에 의존하지 말고, `ptz_service.GetNodes()`를 호출하여 응답이 오는지 확인하는 것이 가장 정확합니다.
3. **프로파일이 없으면 PTZ 제어도 불가:** 설령 PTZ 하드웨어가 있더라도, ONVIF 표준상 PTZ 명령은 특정 `ProfileToken`을 인자로 받아야 하므로 프로파일 조회 문제는 반드시 먼저 해결되어야 합니다.

---------------
## Question
```
카메라 제어 중 오류가 발생했습니다: 'ONVIFCamera' object has no attribute 'devicemgr'

이게 뜸
```
## Answer
이 오류는 사용 중인 **ONVIF 라이브러리 버전에 따라 객체의 내부 속성 이름이 다르거나, 서비스가 아직 초기화되지 않았을 때** 발생합니다.

가장 흔한 이유는 사용 중인 라이브러리가 (현재 거의 표준인) `onvif-zeep`이 아닌 구형 `onvif` 라이브러리이거나, 라이브러리 내부에서 `devicemgr`이라는 이름을 명시적으로 사용하지 않기 때문입니다.

이를 해결할 수 있는 몇 가지 방법입니다.

### 1. `create_devicemgr_service()` 메서드 사용 (가장 확실한 방법)

객체의 속성에 직접 접근하는 대신, 명시적으로 서비스를 생성하여 사용하는 것이 가장 안전합니다. `onvif-zeep` 라이브러리에서는 다음과 같은 패턴을 권장합니다.

```python
# 1. 디바이스 관리 서비스(Device Management Service)를 명시적으로 생성
devicemgr_service = mycam.create_devicemgr_service()

# 2. 생성된 서비스 객체를 통해 기능(Capabilities) 확인
capabilities = devicemgr_service.GetCapabilities({'Category': 'All'})
print(capabilities)

```

### 2. 라이브러리 설치 확인

만약 `pip install onvif` 명령어로 설치하셨다면, 이는 꽤 오래된 구형 버전일 가능성이 높습니다. 최신 환경에서는 `onvif-zeep` 또는 `onvif-zeep-f`를 권장합니다.

* **기존 라이브러리 삭제 후 재설치:**
```bash
pip uninstall onvif
pip install onvif-zeep

```



### 3. `devicemanagement` 속성 확인

일부 라이브러리 버전이나 포크(Fork)된 버전에서는 속성 이름이 `devicemgr`이 아니라 `devicemanagement`로 되어 있는 경우가 있습니다.

```python
# 혹시 이름이 다를 수 있으니 확인
if hasattr(mycam, 'devicemanagement'):
    capabilities = mycam.devicemanagement.GetCapabilities({'Category': 'All'})
elif hasattr(mycam, 'devicemgr'):
    capabilities = mycam.devicemgr.GetCapabilities({'Category': 'All'})

```

### 4. 속성 목록 직접 확인 (디버깅 팁)

객체가 어떤 속성을 가지고 있는지 모를 때는 파이썬의 `dir()` 함수를 사용하여 실시간으로 확인할 수 있습니다.

```python
print(dir(mycam))  # mycam 객체가 가진 모든 속성과 메서드를 출력합니다.

```

출력된 목록 중에 `device`로 시작하는 단어가 있는지 찾아보세요 (예: `devicemanagement`, `devicemgr` 등).

---

**정리하자면:**
이전 답변에서 제가 드린 `mycam.devicemgr.GetCapabilities` 코드가 작동하지 않는다면, 위 1번 방법(`create_devicemgr_service()`)으로 코드를 수정해 보세요. 해당 서비스가 생성되어야 카메라의 PTZ 지원 여부 등을 확인할 수 있는 `GetCapabilities` 함수를 호출할 수 있습니다.

```

```
--------------
## Question
```
카메라 제어 중 오류가 발생했습니다: 'ONVIFCamera' object has no attribute 'create_devicemgr_service'"
이게 뜸. 설치는 pip install onvif-zeep 으로 햇음
```
## Answer
`onvif-zeep` 라이브러리에서 서비스 생성 메서드 명칭이 제가 이전에 제안드린 것과 약간 차이가 있어 발생한 문제입니다. 해당 라이브러리의 표준 메서드 명칭은 **`create_devicemanagement_service`** (mgr이 아닌 전체 이름)입니다.

현재 코드에서 아래와 같이 수정하여 시도해 보시기 바랍니다.

### 1. 정확한 메서드 명칭으로 수정

`devicemagement`라는 전체 이름을 사용해야 합니다.

```python
# 'devicemgr' 대신 'devicemanagement'를 사용
devicemgr_service = mycam.create_devicemanagement_service()
capabilities = devicemgr_service.GetCapabilities({'Category': 'All'})
print(capabilities)

```

### 2. 사용 가능한 메서드 실시간 확인법

라이브러리 버전마다 메서드 이름이 미세하게 다를 수 있습니다. 이럴 때는 파이썬의 **인스펙션(Inspection)** 기능을 사용하여 현재 `mycam` 객체가 실제로 어떤 메서드들을 가지고 있는지 바로 확인할 수 있습니다.

```python
# 'create_'로 시작하는 모든 메서드 목록 출력
print([method for method in dir(mycam) if method.startswith('create_')])

```

이 코드를 실행하면 `['create_devicemanagement_service', 'create_media_service', 'create_ptz_service', ...]`와 같은 리스트가 출력됩니다. 여기에 출력된 이름을 그대로 복사해서 사용하시면 됩니다.

---

### 🔍 왜 이런 문제가 발생하나요?

ONVIF 표준은 여러 개의 서비스(Service)로 나뉘어 있으며, `onvif-zeep`은 각 서비스의 WSDL 파일을 기반으로 동적으로 메서드를 생성합니다.

| 서비스 구분 | 메서드 명칭 (onvif-zeep) | 주요 역할 |
| --- | --- | --- |
| **Device Management** | `create_devicemanagement_service()` | 카메라 정보, 시간, 네트워크, 기능(Capabilities) 확인 |
| **Media** | `create_media_service()` | 프로파일 조회, 비디오 소스 및 스트림 URL 관리 |
| **PTZ** | `create_ptz_service()` | 팬/틸트/줌 제어, 프리셋 설정 |
| **Imaging** | `create_imaging_service()` | 밝기, 대비, 포커스 등 렌즈 설정 |

---

### 💡 팁: 서비스 객체 바로 접근하기

만약 `create_..._service()` 메서드를 호출하는 것이 번거롭다면, `onvif-zeep` 초기화 시점에 이미 생성된 속성이 있는지 확인해 보세요. 보통은 아래와 같이 줄여서 접근할 수도 있습니다.

```python
# 메서드 호출 대신 속성으로 접근 시도 (라이브러리 버전에 따라 다름)
try:
    capabilities = mycam.devicemanagement.GetCapabilities({'Category': 'All'})
except AttributeError:
    # 속성이 없다면 위에서 설명한 create_... 메서드를 사용해야 함
    devicemgr_service = mycam.create_devicemanagement_service()
    capabilities = devicemgr_service.GetCapabilities({'Category': 'All'})

```

위의 1번 방법(`create_devicemanagement_service`)으로 수정하시면 정상적으로 작동할 것입니다. 여전히 오류가 발생한다면 `dir(mycam)`의 출력 결과를 알려주세요. 바로 분석해 드리겠습니다.

----------
## Question
```
여전히 no attribute 가 뜸. 공식 문서를 참고해서 PTZ 지원 여부 확인하는 방법 알려주기 바람.
```
## Answer
계속해서 `AttributeError`가 발생한다면 답답하실 수밖에 없습니다. `onvif-zeep` 라이브러리는 카메라가 제공하는 **WSDL(웹 서비스 정의 언어) 파일을 동적으로 파싱하여 파이썬 객체에 속성을 붙이는(Binding) 방식**으로 동작하기 때문에, 환경이나 버전에 따라 이름이 다르게 매핑되거나 아예 매핑되지 않는 경우가 있습니다.

라이브러리 소스코드 및 ONVIF 공식 규격을 기반으로 가장 확실하게 PTZ 지원 여부를 확인하는 방법을 정리해 드립니다.

---

### 1. 공식 속성명 `devicemgmt` 사용 (가장 표준적인 방법)

`onvif-zeep`은 카메라 객체를 초기화할 때 기본 디바이스 관리 서비스를 **`devicemgmt`** (management의 축약형)라는 이름으로 자동 바인딩합니다. `create_...` 함수를 부를 필요 없이 바로 접근해야 합니다.

```python
from onvif import ONVIFCamera

# 카메라 연결
mycam = ONVIFCamera(cam_ip, cam_port, username, password)

try:
    # devicemgmt 속성을 통해 전체 기능(Capabilities) 요청
    capabilities = mycam.devicemgmt.GetCapabilities({'Category': 'All'})
    
    # PTZ 속성이 존재하는지, 그리고 값이 비어있지 않은지 확인
    if hasattr(capabilities, 'PTZ') and capabilities.PTZ is not None:
        print("✅ PTZ 지원 카메라입니다.")
        print(f"PTZ 엔드포인트 주소: {capabilities.PTZ.XAddr}")
    else:
        print("❌ PTZ를 지원하지 않는 카메라입니다.")

except AttributeError as e:
    print(f"속성 접근 오류: {e}")
except Exception as e:
    print(f"오류 발생: {e}")

```

### 2. `create_ptz_service()`의 예외(Exception)를 활용한 확인

만약 카메라가 아예 PTZ 기능을 하드웨어적으로 또는 펌웨어적으로 제공하지 않는다면, 카메라의 WSDL 응답 트리에 PTZ 서비스 엔드포인트가 빠져 있습니다.
따라서 `create_ptz_service()`를 호출하는 것 자체가 표준적인 지원 여부 검사 방법이 될 수 있습니다.

```python
from onvif import ONVIFCamera
from zeep.exceptions import Fault, Error

mycam = ONVIFCamera(cam_ip, cam_port, username, password)

try:
    ptz_service = mycam.create_ptz_service()
    print("✅ PTZ 서비스 생성 성공. PTZ를 지원합니다.")
    
except AttributeError:
    # 라이브러리가 WSDL에서 PTZ 서비스를 찾지 못해 메서드 자체를 만들지 못한 경우
    print("❌ WSDL에 PTZ 명세가 없습니다. PTZ 미지원 카메라입니다.")
except (Fault, Error) as e:
    # 서비스는 있으나 카메라가 요청을 거부하거나 지원하지 않는 경우
    print(f"❌ PTZ 서비스 통신 실패 (미지원 또는 권한 문제): {e}")
except Exception as e:
    print(f"❌ 알 수 없는 오류: {e}")

```

---

### 💡 그래도 계속 객체 오류가 난다면? (WSDL 경로 문제)

파이썬 개발 환경이나 운영체제에 따라 `onvif-zeep`이 **WSDL 파일을 제대로 찾지 못해서** `devicemgmt`나 다른 서비스 속성을 아예 객체에 붙이지 못하는 경우가 종종 있습니다. (특히 Docker 환경이나 가상 환경에서 자주 발생합니다.)

이때는 카메라 객체를 생성할 때 라이브러리 내부의 WSDL 폴더 경로를 명시적으로 지정해 주어야 합니다.

```python
import os
from onvif import ONVIFCamera

# onvif-zeep 패키지가 설치된 경로 내의 wsdl 폴더를 강제로 지정
# Linux/Mac 가상환경 예시: '/path/to/venv/lib/python3.x/site-packages/wsdl'
wsdl_path = '/usr/local/lib/python3.10/site-packages/wsdl' # 본인의 파이썬 환경에 맞게 수정

mycam = ONVIFCamera(cam_ip, cam_port, username, password, wsdl_path)

```

**디버깅 팁:**
작성하신 스크립트에서 `print(mycam.__dict__.keys())`를 출력해 보시면, 초기화 직후 `mycam` 객체에 실제로 어떤 서비스들(`devicemgmt`, `media` 등)이 바인딩되었는지 정확하게 확인하실 수 있습니다. 만약 이 딕셔너리가 텅 비어있다면 100% WSDL 파싱(경로) 문제입니다.