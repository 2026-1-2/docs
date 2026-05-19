# Hikvision ISAPI를 통한 이벤트 감지 시스템

> **작성일**: 2026-05-19
> **작성자**: lsmin3388
> **프로젝트**: 버티포트 CCTV 감시 시스템

## 1. 개요

버티포트 CCTV 감시 시스템은 Hikvision AcuSense 카메라의 **ISAPI(IP Surveillance API)**를 통해 침입 감지, 라인 크로싱, 영역 진입 등의 스마트 이벤트를 실시간 수신한다. NestJS 백엔드가 `alertStream` 엔드포인트의 `multipart/mixed` 스트림을 파싱하고, 이벤트를 DB에 저장한 뒤 Gmail 알림을 발송한다.

## 2. ISAPI 개요

ISAPI는 Hikvision 독자 RESTful HTTP API로, ONVIF보다 AcuSense 등 고급 기능에 세밀하게 접근할 수 있다.

| 항목 | ISAPI | ONVIF |
|------|-------|-------|
| 표준화 | Hikvision 독자 규격 | 국제 표준 (IEC) |
| 데이터 형식 | XML | SOAP/XML |
| 인증 | Digest Authentication | WS-UsernameToken |
| 스마트 이벤트 | 전체 지원 (AcuSense) | 기본 이벤트만 |
| 실시간 스트림 | alertStream (HTTP long-poll) | PullPoint Subscription |

### 주요 엔드포인트

| 엔드포인트 | 메서드 | 용도 |
|-----------|--------|------|
| `/ISAPI/System/deviceInfo` | GET | 장비 정보 조회 |
| `/ISAPI/Event/notification/alertStream` | GET | 실시간 이벤트 스트림 |
| `/ISAPI/PTZCtrl/channels/1/continuous` | PUT | PTZ 연속 이동 |
| `/ISAPI/Snapshot` | GET | 스냅샷 캡처 |

## 3. Digest Authentication 인증

Hikvision 카메라는 **HTTP Digest Authentication (RFC 7616)**을 사용한다. Basic Auth와 달리 비밀번호가 평문으로 전송되지 않는다.

```
[클라이언트] ── GET /ISAPI/Event/... ──> [카메라]
[클라이언트] <── 401 + WWW-Authenticate: Digest realm="...", nonce="..."
[클라이언트] ── GET + Authorization: Digest response="<MD5 해시>" ──> [카메라]
[클라이언트] <── 200 OK + 이벤트 스트림
```

```typescript
// src/camera/services/isapi.client.ts
import axios from 'axios';
import { createDigestAuth } from '@mreal/digest-auth';

export class IsapiClient {
  private digestAuth = createDigestAuth({ username: this.user, password: this.pass });
  constructor(private host: string, private user: string, private pass: string) {}

  async request(path: string): Promise<any> {
    const url = `http://${this.host}${path}`;
    try { await axios.get(url); }
    catch (err) {
      if (err.response?.status === 401) {
        const auth = this.digestAuth.resolve(err.response.headers['www-authenticate'], 'GET', path);
        return (await axios.get(url, { headers: { Authorization: auth } })).data;
      }
      throw err;
    }
  }
}
```

## 4. alertStream 이벤트 스트림

`GET /ISAPI/Event/notification/alertStream`에 접속하면 카메라가 HTTP 연결을 유지하며 이벤트 발생 시 `multipart/mixed` 형식으로 데이터를 전송한다.

```typescript
// src/camera/services/alert-stream.service.ts
@Injectable()
export class AlertStreamService implements OnModuleInit {
  private readonly xmlParser = new XMLParser({ ignoreAttributes: false });

  async onModuleInit() { this.connect('192.168.1.64', 'admin', 'password123'); }

  private connect(host: string, user: string, pass: string): void {
    const req = http.request({
      hostname: host, path: '/ISAPI/Event/notification/alertStream',
      method: 'GET', auth: `${user}:${pass}`,
    }, (res) => {
      let buffer = '';
      res.on('data', (chunk: Buffer) => {
        buffer += chunk.toString();
        const parts = buffer.split('--boundary');
        buffer = parts.pop() || '';
        for (const part of parts) {
          const m = part.match(/<EventNotificationAlert[\s\S]*?<\/EventNotificationAlert>/);
          if (m) this.handleEvent(m[0]);
        }
      });
      res.on('end', () => setTimeout(() => this.connect(host, user, pass), 5000));
    });
    req.on('error', () => setTimeout(() => this.connect(host, user, pass), 5000));
    req.end();
  }

  private handleEvent(xml: string): void {
    const alert = this.xmlParser.parse(xml).EventNotificationAlert;
    Logger.log(`이벤트: ${alert.eventType} | 채널: ${alert.channelID}`);
  }
}
```

## 5. XML 이벤트 구조 (EventNotificationAlert)

```xml
<EventNotificationAlert version="2.0" xmlns="http://www.hikvision.com/ver20/XMLSchema">
  <ipAddress>192.168.1.64</ipAddress>
  <channelID>1</channelID>
  <dateTime>2026-05-19T14:23:45+09:00</dateTime>
  <eventType>fielddetection</eventType>
  <eventState>active</eventState>
  <DetectionRegionList>
    <DetectionRegionEntry>
      <sensitivityLevel>60</sensitivityLevel>
      <targetType>human</targetType>
    </DetectionRegionEntry>
  </DetectionRegionList>
</EventNotificationAlert>
```

| 필드 | 설명 | 예시 |
|------|------|------|
| `eventType` | 이벤트 유형 | `fielddetection`, `linedetection` |
| `eventState` | 이벤트 상태 | `active`(발생), `inactive`(종료) |
| `targetType` | AcuSense 감지 대상 | `human`, `vehicle` |

## 6. 이벤트 타입 및 AcuSense 필터링

| 이벤트 타입 | 설명 | 용도 |
|------------|------|------|
| `fielddetection` | 영역 침입 감지 | 버티포트 보안 구역 침입 |
| `linedetection` | 라인 크로싱 감지 | 이착륙장 경계선 통과 |
| `regionentrance` | 영역 진입 감지 | 관제 구역 진입 |
| `loitering` | 배회 감지 | 의심 행위 감지 |

AcuSense는 AI 기반으로 감지 대상을 분류한다. `human`/`vehicle`만 처리하고 `other`(동물 등)는 무시하여 오탐을 줄인다.

```typescript
@Injectable()
export class EventFilterService {
  private readonly ALLOWED_TYPES = ['fielddetection', 'linedetection', 'regionentrance', 'loitering'];
  private readonly ALLOWED_TARGETS = ['human', 'vehicle'];

  shouldProcess(event: ParsedEvent): boolean {
    if (event.eventState !== 'active') return false;
    if (!this.ALLOWED_TYPES.includes(event.eventType)) return false;
    if (event.targetType && !this.ALLOWED_TARGETS.includes(event.targetType)) return false;
    return true;
  }
}
```

## 7. 카메라 웹UI 스마트 이벤트 설정

브라우저에서 `http://<카메라IP>` 접속 후 **설정 > 이벤트 > 스마트 이벤트**로 이동한다.

| 설정 항목 | 권장 값 | 설명 |
|-----------|---------|------|
| 영역 설정 | 4각형 그리기 | 버티포트 보안 구역 영역 지정 |
| 감도 | 60 | 0~100, 높을수록 민감 |
| 대상 필터 | 사람, 차량 | AcuSense 필터링 |
| 최소/최대 대상 크기 | 3% / 80% | 화면 대비 크기 |

## 8. 이벤트 DB 저장 (Prisma)

```prisma
model Event {
  id          Int       @id @default(autoincrement())
  cameraId    Int
  camera      Camera    @relation(fields: [cameraId], references: [id])
  eventType   String          // fielddetection, linedetection 등
  targetType  String?         // human, vehicle
  detectedAt  DateTime
  createdAt   DateTime  @default(now())
  snapshotUrl String?
  @@index([cameraId, detectedAt])
}
```

이벤트 저장은 `prisma.event.create({ data })`로 수행한다.

## 9. Gmail 알림 발송

카메라 alertStream 수신 -> XML 파싱 -> AcuSense 필터링 -> MySQL 저장 -> nodemailer로 Gmail 발송.
```typescript
@Injectable()
export class GmailNotificationService {
  private transporter = nodemailer.createTransport({
    service: 'gmail',
    auth: { user: process.env.GMAIL_USER, pass: process.env.GMAIL_APP_PASSWORD },
  });

  async sendAlert(event: { eventType: string; targetType: string; cameraName: string }) {
    const typeKo: Record<string, string> = {
      fielddetection: '영역 침입', linedetection: '라인 크로싱',
      regionentrance: '영역 진입', loitering: '배회 감지',
    };
    await this.transporter.sendMail({
      to: process.env.ALERT_RECIPIENTS,  // admin@knu.ac.kr 등
      subject: `[버티포트 경보] ${typeKo[event.eventType]} 감지`,
      html: `<p>대상: ${event.targetType} | 카메라: ${event.cameraName}</p>`,
    });
  }
}
```

## 10. 요약

- **ISAPI**는 Hikvision 독자 REST API로, AcuSense 스마트 이벤트에 접근할 수 있다.
- **Digest Auth**로 인증하며, **alertStream**에 HTTP long-poll로 `multipart/mixed` 스트림을 수신한다.
- **AcuSense targetType** 필터링으로 사람/차량만 처리하여 오탐을 최소화한다.
- 이벤트는 Prisma로 MySQL에 저장하고, nodemailer로 Gmail 알림을 발송한다.

## 참고 자료

- [Hikvision ISAPI 공식 문서](https://www.hikvision.com/en/support/download/sdk/)
- [RFC 7616 - HTTP Digest Access Authentication](https://datatracker.ietf.org/doc/html/rfc7616)
