# Hikvision 스마트 이벤트 → Gmail 알림 설계

작성일: 2026-05-25

---

## 1. 목표

드론 모니터링 시스템에서 Hikvision `DS-2DE4225IW-DE/T5` PTZ 카메라의 자체 오브젝트 감지(사람 침입) 이벤트를 서버가 수신하고, Gmail로 알림을 자동 발송한다.  
운영자가 화면을 계속 보지 않아도 사람이 감지되면 메일을 받는다.

---

## 2. 카메라 기능 사전 확인 (구현 전 필수)

### 2-1. Human/Vehicle Filter 지원 여부

`DS-2DE4225IW-DE/**T5**` 변형은 AcuSense 계열로 사람·차량 분류 XML을 지원할 가능성이 높다.  
카메라 웹UI에서 직접 확인:

```
카메라 웹UI 접속 → Configuration → Event → Smart Event
→ "Human/Vehicle Filter" 또는 "Target Classification" 옵션 유무 확인
```

| 결과 | 설계 적용 |
|------|-----------|
| 옵션 있음 | XML `<targetType>human</targetType>` 필터링으로 사람만 알림 |
| 옵션 없음 | 침입 감지(`fielddetection`) 이벤트 자체가 트리거 (사람·사물 구분 없음) |

### 2-2. 지원 스마트 이벤트 타입

| eventType 값 | 의미 |
|---|---|
| `fielddetection` | 침입 감지 (지정 구역 진입) |
| `linedetection` | 라인 크로싱 |
| `regionEntrance` | 구역 진입 |
| `regionExiting` | 구역 이탈 |
| `VMD` | 비디오 모션 감지 (기본) |

---

## 3. 전체 아키텍처

```
Hikvision Camera (DS-2DE4225IW-DE/T5)
  │
  │  HTTP POST (multipart/form-data, XML + 선택적 이미지)
  │  → 이벤트 발생 시 카메라가 직접 푸시
  ▼
NestJS (mmp-server)
  └─ POST /alerts/hik-event          ← HikEventController
        │
        ├─ 1. XML 파싱 (EventNotificationAlert)
        ├─ 2. IP 주소로 Camera 레코드 조회 (Prisma)
        ├─ 3. eventType 필터 (fielddetection 등)
        ├─ 4. targetType 필터 (human, 카메라 지원 시)
        ├─ 5. 디바운스 체크 (60초 쿨다운 / 카메라)
        ├─ 6. 스냅샷 캡처 (기존 snapshot() 재사용)
        ├─ 7. Alert DB 저장 (Prisma)
        └─ 8. Gmail 발송 (Nodemailer SMTP + 스냅샷 첨부)
                   │
                   ▼
             운영자 Gmail 수신함
```

### 선택하지 않은 방식: alertStream (Pull)

카메라의 `/ISAPI/Event/notification/alertStream`에 서버가 직접 구독하는 방식.  
이 프로젝트에서 Push를 선택한 이유:
- 기존 카메라당 RTSP 2개 연결이 이미 자원 압박 요인
- alertStream은 카메라에 상시 HTTP 연결을 추가로 맺어야 함
- Push(Webhook)는 이벤트 발생 시만 트래픽 → 카메라 부하 최소
- NestJS 기존 Controller 패턴과 구조적으로 일치

---

## 4. NestJS 구현 설계

### 4-1. 새 모듈 구조

```
src/alerts/
  alerts.module.ts
  alerts.controller.ts    ← POST /alerts/hik-event
  alerts.service.ts       ← 이벤트 처리, 쿨다운, 이메일
  alerts.dto.ts           ← ParsedHikEvent 인터페이스
  email.service.ts        ← Nodemailer 래퍼
```

### 4-2. AlertsController

```typescript
// POST /alerts/hik-event
// @Public() — JWT 가드 제외
// IP 화이트리스트: 카메라 IP만 허용 (Guards 또는 미들웨어)
@Public()
@Post('hik-event')
@UseGuards(CameraIpGuard)
async receiveHikEvent(@Req() req: Request, @Res() res: Response) {
  // multipart or raw XML body 파싱
  // AlertsService.handleEvent(rawBody) 호출
  res.status(200).send();   // 카메라는 200 OK를 기다림
}
```

**주의**: 카메라가 200 OK를 받지 못하면 재전송 루프가 발생한다. 처리 완료 전에 응답해야 한다.

### 4-3. AlertsService 핵심 흐름

```typescript
async handleEvent(rawXml: string, attachedImage?: Buffer) {
  const event = this.parseXml(rawXml);
  // 1. IP → Camera 매핑
  const camera = await this.prisma.camera.findFirst({
    where: { ip_address: event.ipAddress }
  });
  if (!camera) return;  // 미등록 카메라 무시

  // 2. 이벤트 타입 필터
  const watchedTypes = ['fielddetection', 'linedetection', 'regionEntrance'];
  if (!watchedTypes.includes(event.eventType)) return;

  // 3. targetType 필터 (카메라가 지원하는 경우)
  if (event.targetType && event.targetType !== 'human') return;

  // 4. 디바운스 (in-memory cooldown map)
  const cooldownKey = `${camera.camera_id}:${event.eventType}`;
  if (this.isInCooldown(cooldownKey, 60_000)) return;
  this.setCooldown(cooldownKey);

  // 5. 스냅샷 (카메라가 이미지를 미동봉 시 직접 캡처)
  const snapshot = attachedImage ?? await this.camerasService.snapshot(camera.camera_id);

  // 6. DB 저장
  await this.prisma.alert.create({ data: { camera_id: camera.camera_id, event_type: event.eventType, detected_at: new Date() } });

  // 7. Gmail 발송
  await this.emailService.sendAlertEmail(camera, event, snapshot);
}
```

### 4-4. Prisma 스키마 추가

```prisma
model Alert {
  alert_id    Int      @id @default(autoincrement())
  camera_id   Int
  event_type  String
  detected_at DateTime @default(now())
  notified    Boolean  @default(false)
  camera      Camera   @relation(fields: [camera_id], references: [camera_id])
}
```

### 4-5. XML 파싱 대상 필드

Hikvision `EventNotificationAlert` XML 구조:

```xml
<EventNotificationAlert>
  <ipAddress>192.168.0.101</ipAddress>
  <macAddress>XX:XX:XX:XX:XX:XX</macAddress>
  <channelID>1</channelID>
  <dateTime>2026-05-25T14:30:00+09:00</dateTime>
  <eventType>fielddetection</eventType>
  <eventState>active</eventState>   <!-- active | inactive -->
  <eventDescription>intrusion detection</eventDescription>
  <!-- AcuSense/T5 변형에서 추가 -->
  <DetectionRegionList>
    <DetectionRegionEntry>
      <targetType>human</targetType>
    </DetectionRegionEntry>
  </DetectionRegionList>
</EventNotificationAlert>
```

`xml2js` 또는 `fast-xml-parser` 패키지로 파싱.

---

## 5. Gmail 설정 및 Nodemailer

### 5-1. Gmail App Password 설정 (필수)

Google이 "보안 수준이 낮은 앱" 접근을 폐기했으므로 App Password가 필수다.

```
1. Gmail 계정 → Google 계정 관리
2. 보안 → 2단계 인증 활성화 (미활성 시 App Password 메뉴 없음)
3. 보안 → 앱 비밀번호 → 앱: 메일, 기기: 기타(이름 임의) → 생성
4. 생성된 16자리 코드를 .env에 저장
```

### 5-2. 환경 변수 (.env)

```env
GMAIL_USER=your-address@gmail.com
GMAIL_APP_PASSWORD=xxxx xxxx xxxx xxxx   # 16자리 앱 비밀번호
ALERT_RECIPIENT=recipient@gmail.com       # 알림 수신 주소
ALERT_COOLDOWN_SEC=60                     # 쿨다운 (초)
```

### 5-3. Nodemailer 설정

```typescript
// email.service.ts
const transporter = nodemailer.createTransport({
  host: 'smtp.gmail.com',
  port: 465,
  secure: true,          // SSL
  auth: {
    user: process.env.GMAIL_USER,
    pass: process.env.GMAIL_APP_PASSWORD,
  },
});
```

### 5-4. 메일 본문 예시

```
제목: [경고] CAM-01 활주로 — 사람 감지 (2026-05-25 14:30)
본문:
  카메라: CAM-01 활주로 (192.168.0.101)
  감지 유형: 침입 감지 (fielddetection)
  감지 시각: 2026-05-25 14:30:05
  [첨부: snapshot.jpg]
```

---

## 6. 카메라 웹UI 설정 방법

카메라 측에서 NestJS 서버를 HTTP 알람 수신지로 등록해야 한다.

```
카메라 웹UI 접속 (http://[카메라IP])
→ Configuration
→ Network
→ Advanced Settings
→ HTTP Listening (또는 Alarm Server)
   - IP/Domain: [NestJS 서버 IP]
   - Port: [NestJS 포트, 기본 3000]
   - URL: /alerts/hik-event
   - Protocol: HTTP
   - (선택) Username/Password: 기본 인증 설정 가능

→ Smart Event 설정 (Configuration → Event → Smart Event)
   - Intrusion Detection 활성화
   - 감지 구역 그리기
   - Linkage Method → HTTP Listening 체크
```

---

## 7. 보안 고려사항

| 항목 | 방법 |
|------|------|
| 웹훅 엔드포인트 인증 | `@Public()` + `CameraIpGuard` (허용 IP 목록과 요청 IP 비교) |
| App Password 노출 방지 | `.env`에만 저장, `.gitignore` 확인 |
| Gmail 일일 한도 | 개인 Gmail ~500통/일. 쿨다운 60초 시 카메라 1대 기준 최대 1,440통/일 → **쿨다운 필수** |
| 재전송 루프 방지 | 카메라에 무조건 HTTP 200 즉시 응답 |

---

## 8. 구현 순서

1. **카메라 웹UI 확인**: Smart Event 탭에서 사람/차량 필터 지원 여부 확인
2. **Prisma schema 추가**: `Alert` 모델 추가 및 마이그레이션
3. **패키지 설치**: `npm i fast-xml-parser nodemailer` + `@types/nodemailer`
4. **AlertsModule 구현**: Controller → Service → EmailService 순서
5. **카메라 웹UI 설정**: HTTP Listening 서버로 NestJS 등록
6. **로컬 테스트**: `curl -X POST http://localhost:3000/alerts/hik-event` + 테스트 XML로 이메일 발송 확인
7. **카메라 트리거 테스트**: 실제로 카메라 앞에서 움직여 Gmail 수신 확인

---

## 9. 미결 사항

- [ ] 카메라 웹UI에서 Human Filter 지원 여부 확인 (2-1 참고)
- [ ] Alert 히스토리를 대시보드에도 표시할지 여부 (WebSocket 추가 여부)
- [ ] 여러 수신자(팀원 전체)에게 발송할지 여부
