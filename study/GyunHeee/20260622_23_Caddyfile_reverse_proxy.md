# Caddyfile에서 경로별 reverse_proxy 설정

## 전체 구조 목표

```
브라우저 → Caddy (HTTPS) → 경로에 따라 분기
    /api/*      → mmp-server:3000  (NestJS)
    /streams/*  → mediamtx:8888   (HLS 스트리밍)
    /*          → react-app:80    (React 정적 파일)
```

---

## Caddyfile 작성

```caddyfile
your-domain.com {

    # NestJS API
    handle /api/* {
        reverse_proxy mmp-server:3000
    }

    # MediaMTX HLS 스트리밍
    handle /streams/* {
        reverse_proxy mediamtx:8888
    }

    # MediaMTX WebRTC
    handle /webrtc/* {
        reverse_proxy mediamtx:8889
    }

    # React 앱 (catch-all, 반드시 마지막에)
    handle {
        reverse_proxy react-app:80
    }
}
```

`handle`은 경로 매칭 시 다른 handle과 상호 배타적이다. `route`와 달리 순서에 덜 민감하다.

---

## handle vs handle_path 차이

```caddyfile
# handle: 경로 그대로 upstream에 전달
handle /api/* {
    reverse_proxy mmp-server:3000
    # /api/cameras → mmp-server:3000/api/cameras
}

# handle_path: 매칭된 prefix를 제거하고 전달
handle_path /api/* {
    reverse_proxy mmp-server:3000
    # /api/cameras → mmp-server:3000/cameras  (prefix 제거)
}
```

NestJS가 `/api` prefix 없이 라우팅한다면 `handle_path`를 사용한다.

---

## MediaMTX HLS 경로 상세 설정

```caddyfile
handle /streams/* {
    # 경로에서 /streams 제거 후 MediaMTX로 전달
    uri strip_prefix /streams

    reverse_proxy mediamtx:8888 {
        header_up Host {upstream_hostport}
    }
}
```

MediaMTX HLS는 기본적으로 `/{path}/index.m3u8` 형태로 요청을 받는다.
경로 설계에 따라 prefix 제거 여부를 결정한다.

---

## docker-compose.yml과 함께 사용

```yaml
services:
  caddy:
    image: caddy:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy-data:/data      # 인증서 저장
      - caddy-config:/config
    networks:
      - mmp-network

  mmp-server:
    networks:
      - mmp-network
    # ports 없음 → Caddy만 접근 가능

  mediamtx:
    networks:
      - mmp-network
    ports:
      - "8554:8554"   # RTSP는 Caddy 우회, 직접 노출
    # 8888(HLS), 8889(WebRTC)는 Caddy를 통해 HTTPS로 서빙

volumes:
  caddy-data:
  caddy-config:
```

---

## 주의: RTSP는 Caddy로 프록시할 수 없다

RTSP는 TCP 기반이지만 HTTP가 아니다. Caddy는 HTTP(S) 리버스 프록시다.
RTSP 포트(8554)는 Caddy를 거치지 않고 직접 노출해야 한다.
HLS/WebRTC는 HTTP 기반이므로 Caddy를 통해 HTTPS로 서빙 가능하다.
