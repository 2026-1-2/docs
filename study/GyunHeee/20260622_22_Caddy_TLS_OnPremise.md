# Caddy 자동 TLS가 온프레미스 환경에서 동작하는 방식

## Let's Encrypt HTTP-01 Challenge의 전제 조건

Caddy가 자동으로 인증서를 발급받으려면 Let's Encrypt가 외부에서 해당 도메인으로 HTTP 요청을 보낼 수 있어야 한다.

```
Let's Encrypt 서버
      ↓ HTTP GET http://your-domain.com/.well-known/acme-challenge/{token}
  온프레미스 서버 (Caddy)
      ↓ 응답
Let's Encrypt가 도메인 소유 확인 → 인증서 발급
```

**조건**: 외부 인터넷 → 서버 80번 포트가 열려 있어야 한다.

---

## 온프레미스 환경에서의 문제

사내 서버는 보통 공인 IP가 없거나, 방화벽으로 외부에서 접근이 차단된다.

```
인터넷 → 공유기/방화벽 → 사내 서버 (사설 IP: 192.168.x.x)
```

Let's Encrypt가 `192.168.x.x`로는 접근할 수 없다.

---

## 해결 방법

### 방법 1: 공인 IP + 포트 포워딩 (가장 단순)

```
인터넷 → 공인 IP:80 → 공유기 포트 포워딩 → 사내 서버:80
```

공유기에서 80번, 443번 포트를 사내 서버로 포워딩하면 HTTP-01 challenge 통과 가능.
도메인은 공인 IP에 A레코드로 연결.

```
# Caddyfile
your-domain.com {
    # 자동으로 Let's Encrypt 인증서 발급
    reverse_proxy mmp-server:3000
}
```

### 방법 2: DNS-01 Challenge (외부 접근 없이 가능)

DNS 레코드로 도메인 소유를 증명하는 방식. 80포트를 열지 않아도 된다.

```
# Caddyfile
your-domain.com {
    tls {
        dns cloudflare {env.CLOUDFLARE_API_TOKEN}
    }
    reverse_proxy mmp-server:3000
}
```

DNS 공급자 플러그인이 필요하다 (Cloudflare, Route53 등).
Caddy 공식 빌드에는 포함되어 있지 않아 커스텀 빌드가 필요하다.

```dockerfile
# DNS 플러그인 포함 Caddy 빌드
FROM caddy:builder AS builder
RUN xcaddy build --with github.com/caddy-dns/cloudflare

FROM caddy:latest
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### 방법 3: 자체 서명 인증서 (내부망 전용)

외부 접근이 완전히 필요 없는 경우.

```
# Caddyfile
your-domain.com {
    tls internal   # Caddy가 자체 CA로 인증서 생성
    reverse_proxy mmp-server:3000
}
```

브라우저에서 "신뢰할 수 없는 인증서" 경고가 뜨므로, 클라이언트에 Caddy의 root CA를 신뢰 목록에 추가해야 한다.

---

## 온프레미스 프로젝트 권장 선택

| 상황 | 방법 |
|------|------|
| 공인 IP 있고 포트 열 수 있음 | HTTP-01 (기본 Caddy) |
| 포트 못 열지만 DNS 관리 가능 | DNS-01 (플러그인 빌드) |
| 완전 내부망, 외부 접근 없음 | `tls internal` |
