# WebSocket/SSE가 Caddy를 통과할 때 신경 써야 할 설정

## Caddy의 기본 동작

Caddy는 WebSocket과 SSE를 별도 설정 없이 대부분 자동으로 처리한다.
HTTP/1.1 Upgrade 헤더(WebSocket)와 `text/event-stream`(SSE) 모두 기본 지원.

그러나 몇 가지 상황에서 명시적 설정이 필요하다.

---

## WebSocket 설정

```caddyfile
your-domain.com {
    handle /socket.io/* {
        reverse_proxy mmp-server:3000 {
            # WebSocket upgrade 헤더 전달
            header_up Connection {http.request.header.Connection}
            header_up Upgrade {http.request.header.Upgrade}
        }
    }
}
```

Caddy 2.x는 WebSocket upgrade를 자동으로 감지하므로 대부분 위 설정 없이도 동작한다.
문제가 생길 때 명시적으로 추가한다.

---

## SSE 타임아웃 설정 (핵심)

SSE는 연결을 **계속 열어두는** 방식이다.
Caddy의 기본 타임아웃이 SSE 연결을 중간에 끊어버릴 수 있다.

```caddyfile
your-domain.com {
    handle /alerts/stream {
        reverse_proxy mmp-server:3000 {
            # 응답 타임아웃 비활성화 (SSE는 계속 열려있음)
            flush_interval -1

            # transport 타임아웃 설정
            transport http {
                read_timeout 0       # 읽기 타임아웃 없음
                write_timeout 0      # 쓰기 타임아웃 없음
                response_header_timeout 30s
            }
        }
    }
}
```

`flush_interval -1`이 가장 중요하다.
기본값은 Caddy가 응답을 버퍼링해서 일정 크기가 되면 클라이언트에 보내는데,
SSE는 각 이벤트를 즉시 전달해야 하므로 버퍼링 없이 즉시 flush 해야 한다.

---

## HLS 스트리밍 설정

HLS는 `.m3u8` 플레이리스트와 `.ts` 세그먼트 파일 요청이 반복적으로 발생한다.

```caddyfile
handle /streams/* {
    reverse_proxy mediamtx:8888 {
        flush_interval -1   # 세그먼트 파일 즉시 전달

        header_up Host {upstream_hostport}

        # CORS 헤더 (React에서 다른 도메인으로 요청할 경우)
        header_down Access-Control-Allow-Origin *
    }
}
```

---

## 헤더 전달 설정

Caddy가 프록시할 때 기본으로 추가하는 헤더:

```
X-Forwarded-For: 클라이언트 IP
X-Forwarded-Proto: https
X-Forwarded-Host: your-domain.com
```

NestJS에서 실제 클라이언트 IP를 알려면:

```typescript
// main.ts
app.set('trust proxy', 1);  // Caddy 뒤에서 X-Forwarded-For 신뢰
```

---

## 전체 Caddyfile 예시 (SSE + WebSocket + HLS 통합)

```caddyfile
your-domain.com {

    # SSE 알람 스트림
    handle /alerts/stream {
        reverse_proxy mmp-server:3000 {
            flush_interval -1
            transport http {
                read_timeout 0
                write_timeout 0
            }
        }
    }

    # HLS 스트리밍
    handle /streams/* {
        reverse_proxy mediamtx:8888 {
            flush_interval -1
        }
    }

    # NestJS API (일반 HTTP)
    handle /api/* {
        reverse_proxy mmp-server:3000
    }

    # React
    handle {
        reverse_proxy react-app:80
    }
}
```

---

## 문제 증상으로 원인 찾기

| 증상 | 원인 | 해결 |
|------|------|------|
| SSE 연결이 30초마다 끊김 | 타임아웃 | `read_timeout 0` |
| SSE 이벤트가 뭉쳐서 옴 | 버퍼링 | `flush_interval -1` |
| WebSocket 101 업그레이드 실패 | 헤더 누락 | `header_up Upgrade` |
| HLS 영상이 버벅임 | 세그먼트 버퍼링 | `flush_interval -1` |
