# 프롬프트
> Hikvision alertStream을 Python과 NestJS 각각으로 구현할 때 동작 방식이 어떻게 다른지 비교해줘

## 같은 목표, 다른 실행 모델

alertStream은 카메라와 HTTP 연결을 계속 열어두고 데이터를 읽는 **장기 실행 I/O 작업**이다.  
Python과 NestJS가 이 작업을 어떻게 처리하는지 비교한다.

---

## Python 구현

```python
import threading
import requests
from requests.auth import HTTPDigestAuth

class AlertStreamListener(threading.Thread):
    def run(self):
        resp = requests.get(
            'http://192.168.0.101/ISAPI/Event/notification/alertStream',
            auth=HTTPDigestAuth('admin', 'pw'),
            stream=True,        # 연결을 끊지 않고 스트리밍
            timeout=(10, None)  # 연결 타임아웃 10s, 읽기는 무한
        )
        for line in resp.iter_lines(decode_unicode=True):
            # iter_lines()는 줄이 올 때까지 여기서 블로킹
            # 하지만 다른 스레드는 계속 실행 중
            self.parse(line)

t = AlertStreamListener(daemon=True)
t.start()
# 메인 스레드는 계속 실행
```

**실행 흐름:**

```
메인 스레드:  ──────────────────────────────────────────►
                                                        (다른 작업 가능)

리스너 스레드: ──[연결]──[대기]──[XML 수신]──[파싱]──[대기]──►
                           ↑ OS가 네트워크 데이터 대기
```

- `iter_lines()`는 동기 호출이지만 스레드가 분리돼 있어 블로킹이 다른 스레드에 영향 없음
- 카메라 2대 → 스레드 2개로 자연스럽게 확장

---

## NestJS 구현

```typescript
import { Injectable, OnModuleInit } from '@nestjs/common';
import axios from 'axios';

@Injectable()
export class AlertsService implements OnModuleInit {
  onModuleInit() {
    this.startStream();  // await 없이 호출 → 백그라운드 실행
  }

  private async startStream() {
    const resp = await axios.get(
      'http://192.168.0.101/ISAPI/Event/notification/alertStream',
      {
        responseType: 'stream',
        auth: { username: 'admin', password: 'pw' },
      }
    );

    // resp.data는 Node.js Readable 스트림
    resp.data.on('data', (chunk: Buffer) => {
      const line = chunk.toString();
      this.parse(line);
    });
  }
}
```

**실행 흐름:**

```
이벤트 루프:  ──[요청A]──[요청B]──[스트림 데이터 수신 콜백]──[요청C]──►
                                         ↑
                           네트워크 I/O는 libuv가 처리
                           데이터 도착 시 콜백 큐에 추가
```

- 스레드가 아닌 이벤트 기반 콜백
- 스트림 데이터가 없으면 콜백이 실행되지 않아 CPU 낭비 없음
- 파싱 로직이 오래 걸리면 이벤트 루프가 그만큼 멈춤 (주의)

---

## 핵심 차이 비교

| 항목 | Python (threading) | NestJS (이벤트 루프) |
|------|-------------------|-------------------|
| 동시성 방식 | OS 스레드 | 단일 스레드 이벤트 루프 |
| 스트림 읽기 | `iter_lines()` 블로킹 (스레드 내부) | `data` 이벤트 콜백 (논블로킹) |
| 카메라 추가 | 스레드 1개 추가 | 비동기 함수 1개 추가 |
| 메모리 | 스레드당 ~1-8MB 스택 | 콜백만 추가, 스택 없음 |
| 에러 격리 | 스레드 크래시 → 해당 카메라만 영향 | 예외 처리 안 하면 전체 영향 |
| 라이브러리 | `requests` (검증된 동기 라이브러리) | `axios` stream 또는 `node:http` |
| 구현 난이도 | 상대적으로 단순 | multipart 파싱 직접 구현 필요 |

---

## multipart 파싱 차이

alertStream 응답은 `multipart/mixed` 형식이다.  
경계선(`boundary`)으로 XML 블록이 구분된다.

**Python — 줄 단위로 자연스럽게 처리:**

```python
buf = ''
for line in resp.iter_lines(decode_unicode=True):
    if '<EventNotificationAlert' in line:
        buf = line
    elif '</EventNotificationAlert>' in line:
        buf += line
        handle(ET.fromstring(buf))
        buf = ''
    elif buf:
        buf += line
```

**NestJS — chunk 단위로 쌓아서 처리 (더 복잡):**

```typescript
let buf = '';
resp.data.on('data', (chunk: Buffer) => {
  buf += chunk.toString();
  // chunk 경계가 XML 태그 중간에 걸릴 수 있음
  const start = buf.indexOf('<EventNotificationAlert');
  const end = buf.indexOf('</EventNotificationAlert>');
  if (start !== -1 && end !== -1) {
    const xml = buf.slice(start, end + '</EventNotificationAlert>'.length);
    handle(xml);
    buf = buf.slice(end + '</EventNotificationAlert>'.length);
  }
});
```

chunk가 XML 경계 중간에서 잘릴 수 있어 버퍼 관리가 필요하다.  
Python의 `iter_lines()`는 이를 자동으로 처리해준다.

---

## 결론: 이 프로젝트에서의 선택

```
alertStream 구독 → Python이 단순
  - requests + iter_lines() 로 검증된 패턴
  - pyHik 오픈소스 참고 구현 존재
  - threading으로 카메라별 격리

NestJS가 유리한 부분
  - 기존 Camera DB(Prisma)와 직접 연결
  - JWT 인증, 기존 모듈과 통합
  - 대시보드 WebSocket 알림 추가 시 자연스러움

→ 현재 선택: Python 별도 모듈로 alertStream 구독 + Gmail 발송
  NestJS와 연동 필요 시 HTTP API로 카메라 목록 조회
```
