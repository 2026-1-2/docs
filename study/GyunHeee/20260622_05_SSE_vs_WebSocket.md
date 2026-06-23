# NestJS에서 SSE 구현 및 WebSocket과의 비교

## SSE vs WebSocket 선택 기준

| | SSE | WebSocket |
|--|-----|-----------|
| 방향 | 서버 → 클라이언트 (단방향) | 양방향 |
| 프로토콜 | HTTP | ws:// |
| 재연결 | 브라우저 자동 처리 | 직접 구현 필요 |
| 인프라 | 기존 HTTP 인프라 사용 가능 | 별도 설정 필요 (nginx 등) |
| 복잡도 | 낮음 | 높음 |
| 용도 | 알람, 상태 업데이트, 이벤트 알림 | 채팅, 게임, 실시간 협업 |

---

## 카메라 모니터링에 SSE를 선택하는 이유

카메라 스트림 상태 변경, 알람 발생, 연결 끊김 알림 등은 **서버가 클라이언트에게 일방적으로 푸시**하는 패턴이다.
클라이언트가 서버에게 보낼 데이터가 없다. SSE로 충분하다.

WebSocket은 양방향이 필요할 때만 쓴다. 과도한 설계다.

---

## NestJS SSE 구현

```typescript
// alert.controller.ts
import { Controller, Get, Sse, MessageEvent, Param } from '@nestjs/common';
import { Observable } from 'rxjs';
import { AlertService } from './alert.service';

@Controller('alerts')
export class AlertController {
  constructor(private readonly alertService: AlertService) {}

  @Sse('stream')
  streamAlerts(): Observable<MessageEvent> {
    return this.alertService.getAlertStream();
  }
}
```

```typescript
// alert.service.ts
import { Injectable } from '@nestjs/common';
import { Subject, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

@Injectable()
export class AlertService {
  private readonly alertSubject = new Subject<any>();

  getAlertStream(): Observable<MessageEvent> {
    return this.alertSubject.asObservable().pipe(
      map((alert) => ({ data: alert })),
    );
  }

  // webhook이나 내부 이벤트에서 호출
  pushAlert(alert: any) {
    this.alertSubject.next(alert);
  }
}
```

---

## 클라이언트(React)에서 수신

```javascript
const eventSource = new EventSource('/alerts/stream', {
  withCredentials: true,
});

eventSource.onmessage = (event) => {
  const alert = JSON.parse(event.data);
  console.log('새 알람:', alert);
};

eventSource.onerror = () => {
  // 브라우저가 자동으로 재연결 시도
};
```

---

## MediaMTX webhook → SSE 연결

```
MediaMTX → POST /webhook/stream-down → NestJS
                                          ↓
                                    alertService.pushAlert()
                                          ↓
                                    Subject.next()
                                          ↓
                                    React SSE 수신
```

webhook을 받아서 Subject에 넣으면, 구독 중인 모든 클라이언트에게 실시간으로 전달된다.
