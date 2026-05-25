# Hikvision alertStream 수신 → Gmail 알림 구현 설계

작성일: 2026-05-25

---

## 1. 개요

서버가 카메라의 `/ISAPI/Event/notification/alertStream` 엔드포인트에 GET 요청을 보내고,  
카메라가 이벤트 발생 시 XML을 스트리밍으로 흘려보내는 **Pull 방식**을 사용한다.

```
NestJS / Python
  └─ GET ──────────────────────────────────► http://[카메라IP]/ISAPI/Event/notification/alertStream
                                              (Digest Auth)
     ◄─────────────────────────────────────  multipart/mixed 스트림
                                              (이벤트 발생할 때마다 XML 블록 전송)
```

---

## 2. STEP 0 — curl로 실제 카메라 응답 먼저 확인 (구현 전 필수)

### 2-1. 명령어

```bash
curl -v --digest -u admin:비밀번호 \
  http://[카메라IP]/ISAPI/Event/notification/alertStream
```

### 2-2. 응답 해석

**정상 응답 — 스트리밍 시작:**

```
< HTTP/1.1 200 OK
< Content-Type: multipart/mixed; boundary=<boundary>
```

이후 이벤트가 발생할 때마다 아래 형태의 XML 블록이 출력된다:

```
--<boundary>
Content-Type: application/xml; charset=UTF-8

<?xml version="1.0" encoding="UTF-8"?>
<EventNotificationAlert xmlns="http://www.hikvision.com/ver20/XMLSchema">
  <ipAddress>192.168.0.101</ipAddress>
  <portNo>80</portNo>
  <channelID>1</channelID>
  <dateTime>2026-05-25T14:30:05+09:00</dateTime>
  <activePostCount>1</activePostCount>
  <eventType>fielddetection</eventType>
  <eventState>active</eventState>
  <eventDescription>fielddetection</eventDescription>
</EventNotificationAlert>
```

**카메라가 AcuSense(T5 변형)라면 추가 필드가 붙을 수 있음:**

```xml
<DetectionRegionList>
  <DetectionRegionEntry>
    <regionID>1</regionID>
    <targetType>human</targetType>   ← 사람/차량 분류 지원 시
  </DetectionRegionEntry>
</DetectionRegionList>
```

**오류 응답:**

| HTTP 상태 | 원인 |
|-----------|------|
| `401 Unauthorized` | 계정/비밀번호 오류 또는 Digest Auth 미지원 |
| `404 Not Found` | 해당 펌웨어가 alertStream 미지원 |
| `200` + 빈 스트림 | 스마트 이벤트 미활성화 → 카메라 웹UI에서 활성화 필요 |

### 2-3. curl 결과에 따른 분기

```
curl 결과
  ├─ 200 + XML 수신
  │    ├─ <targetType>human</targetType> 있음 → 사람 감지 필터링 가능 ✓
  │    └─ <targetType> 없음              → fielddetection 이벤트 자체가 트리거
  └─ 404 / 연결 불가 → 이 방식 사용 불가, 카메라 웹UI HTTP Listening 설정 검토
```

---

## 3. 카메라 사전 설정 (웹UI)

curl 전에 카메라에서 스마트 이벤트를 활성화해야 스트림에 데이터가 흐른다.

```
카메라 웹UI 접속 (http://[카메라IP])
→ Configuration
→ Event
→ Smart Event
  → Intrusion Detection (침입 감지) 활성화
  → 감지 구역 그리기
  → Arming Schedule 설정 (24시간 or 원하는 시간대)
  → Linkage Method: 체크 항목 없어도 됨 (alertStream은 별도 설정 불필요)
→ Save
```

---

## 4. 구현 위치 결정

| 서버 | 장점 | 단점 |
|------|------|------|
| **Python (video-recorder)** | `requests` 라이브러리로 alertStream 구독이 가장 단순. 이미 스레드 루프 구조 존재. `smtplib`으로 Gmail 발송 간단. | 현재 HTTP 서버가 없어 단독 프로세스로 실행 |
| **NestJS (mmp-server)** | 기존 카메라 DB(`Camera.ip_address`)와 바로 연결 가능 | Node.js에서 multipart 스트림 파싱이 상대적으로 복잡 |

**권장: Python 신규 모듈로 분리**  
video-recorder와 같은 컨테이너 또는 별도 컨테이너로 실행.  
NestJS DB와 연동이 필요하면 HTTP API로 카메라 정보를 조회하는 방식으로 연결.

---

## 5. Python 구현 설계

### 5-1. 파일 구조 (신규 모듈)

```
hik-event-listener/
  main.py              ← 진입점, 카메라별 스레드 실행
  listener.py          ← alertStream 구독 + XML 파싱
  notifier.py          ← Gmail 발송
  cooldown.py          ← 디바운스 (60초 쿨다운)
  config.py            ← 환경변수 로드
  requirements.txt
  Dockerfile
  .env
```

### 5-2. listener.py 핵심 로직

```python
import requests
from requests.auth import HTTPDigestAuth
import xml.etree.ElementTree as ET
import threading

WATCHED_EVENTS = {'fielddetection', 'linedetection', 'regionentrance', 'regionexiting'}
NS = 'http://www.hikvision.com/ver20/XMLSchema'

class AlertStreamListener(threading.Thread):
    def __init__(self, camera_ip, username, password, on_event):
        super().__init__(daemon=True)
        self.url = f'http://{camera_ip}/ISAPI/Event/notification/alertStream'
        self.auth = HTTPDigestAuth(username, password)
        self.on_event = on_event   # 이벤트 발생 시 호출할 콜백

    def run(self):
        while True:
            try:
                resp = requests.get(self.url, auth=self.auth, stream=True, timeout=(10, None))
                self._parse_stream(resp)
            except Exception as e:
                print(f'Stream error: {e}, reconnecting in 10s...')
                time.sleep(10)   # 재연결

    def _parse_stream(self, resp):
        buf = ''
        for line in resp.iter_lines(decode_unicode=True):
            if '<EventNotificationAlert' in line:
                buf = line
            elif '</EventNotificationAlert>' in line:
                buf += line
                self._handle_xml(buf)
                buf = ''
            elif buf:
                buf += line

    def _handle_xml(self, xml_str):
        tree = ET.fromstring(xml_str)

        def find(tag):
            return tree.find(f'{{{NS}}}{tag}')

        event_type = (find('eventType') or {}).text or ''
        event_state = (find('eventState') or {}).text or ''
        channel_id = (find('channelID') or {}).text or '1'

        if event_type.lower() not in WATCHED_EVENTS:
            return
        if event_state != 'active':
            return

        # AcuSense 카메라: targetType 필터
        target_entry = tree.find(f'.//{{{NS}}}DetectionRegionEntry')
        if target_entry is not None:
            target_type = target_entry.find(f'{{{NS}}}targetType')
            if target_type is not None and target_type.text != 'human':
                return   # 차량 등 비인간 무시

        self.on_event({
            'event_type': event_type,
            'channel_id': channel_id,
        })
```

### 5-3. cooldown.py

```python
import time

class CooldownMap:
    def __init__(self, seconds=60):
        self._map = {}
        self._cooldown = seconds

    def is_blocked(self, key: str) -> bool:
        last = self._map.get(key, 0)
        return (time.time() - last) < self._cooldown

    def mark(self, key: str):
        self._map[key] = time.time()
```

### 5-4. notifier.py

```python
import smtplib
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.image import MIMEImage
from datetime import datetime

class GmailNotifier:
    def __init__(self, sender, app_password, recipient):
        self.sender = sender
        self.app_password = app_password
        self.recipient = recipient

    def send(self, camera_ip: str, event_type: str, snapshot: bytes | None = None):
        msg = MIMEMultipart()
        msg['Subject'] = f'[경고] {camera_ip} — 사람 감지 ({datetime.now().strftime("%Y-%m-%d %H:%M")})'
        msg['From'] = self.sender
        msg['To'] = self.recipient

        body = (
            f'카메라 IP: {camera_ip}\n'
            f'감지 유형: {event_type}\n'
            f'감지 시각: {datetime.now().strftime("%Y-%m-%d %H:%M:%S")}'
        )
        msg.attach(MIMEText(body, 'plain'))

        if snapshot:
            img = MIMEImage(snapshot, name='snapshot.jpg')
            msg.attach(img)

        with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp:
            smtp.login(self.sender, self.app_password)
            smtp.sendmail(self.sender, self.recipient, msg.as_string())
```

### 5-5. main.py

```python
import os
from listener import AlertStreamListener
from notifier import GmailNotifier
from cooldown import CooldownMap

CAMERAS = [
    {'ip': os.environ['CAM1_IP'], 'user': os.environ['CAM1_USER'], 'password': os.environ['CAM1_PASSWORD']},
    # 카메라 추가 시 여기에 항목 추가
]

cooldown = CooldownMap(seconds=int(os.environ.get('COOLDOWN_SEC', 60)))
notifier = GmailNotifier(
    sender=os.environ['GMAIL_USER'],
    app_password=os.environ['GMAIL_APP_PASSWORD'],
    recipient=os.environ['ALERT_RECIPIENT'],
)

def on_event(camera_ip, event):
    key = f'{camera_ip}:{event["event_type"]}'
    if cooldown.is_blocked(key):
        return
    cooldown.mark(key)
    print(f'[ALERT] {camera_ip} → {event["event_type"]}')
    notifier.send(camera_ip, event['event_type'])

threads = []
for cam in CAMERAS:
    t = AlertStreamListener(
        camera_ip=cam['ip'],
        username=cam['user'],
        password=cam['password'],
        on_event=lambda ev, ip=cam['ip']: on_event(ip, ev),
    )
    t.start()
    threads.append(t)

for t in threads:
    t.join()
```

### 5-6. .env

```env
CAM1_IP=192.168.0.101
CAM1_USER=admin
CAM1_PASSWORD=카메라비밀번호

GMAIL_USER=your@gmail.com
GMAIL_APP_PASSWORD=xxxx xxxx xxxx xxxx   # Gmail 앱 비밀번호 (16자리)
ALERT_RECIPIENT=recipient@gmail.com
COOLDOWN_SEC=60
```

### 5-7. requirements.txt

```
requests==2.32.3
```

---

## 6. Gmail App Password 설정 (필수)

```
1. Gmail 계정 → Google 계정 관리 → 보안
2. 2단계 인증 활성화
3. 보안 → 앱 비밀번호 → 앱: 메일, 기기: 기타 → 생성
4. 생성된 16자리 코드 → .env의 GMAIL_APP_PASSWORD에 입력
```

SMTP 설정: `smtp.gmail.com:465` (SSL), 일일 발송 한도 ~500통.  
쿨다운 60초 기준 카메라 1대 최대 1,440통/일 → 쿨다운 필수.

---

## 7. 구현 순서

```
1. curl 테스트
   → curl -v --digest -u admin:pw http://[카메라IP]/ISAPI/Event/notification/alertStream
   → XML 응답 확인, targetType 유무 확인

2. 카메라 웹UI에서 Smart Event (Intrusion Detection) 활성화

3. Gmail App Password 발급

4. .env 작성

5. listener.py / notifier.py / cooldown.py / main.py 구현

6. python main.py 로컬 실행 후 카메라 앞에서 움직여 Gmail 수신 확인

7. Docker 컨테이너화 (필요 시)
```

---

## 8. 미결 사항

- [ ] curl 테스트로 실제 XML 구조 및 `<targetType>` 유무 확인
- [ ] NestJS Camera DB와 연동 여부 (필요 시 HTTP API로 카메라 목록 조회)
- [ ] 스냅샷 첨부 여부 (`ffmpeg -i rtsp://... -vframes 1 -f image2 pipe:1` 재사용 가능)
- [ ] Docker Compose에 기존 서비스와 함께 추가할지 여부
