# Docker 컨테이너 간 네트워크 구성과 포트 노출 기준

## 기본 개념

Docker Compose는 기본적으로 모든 서비스를 같은 브리지 네트워크에 올린다.
같은 네트워크 안에서는 서비스 이름으로 통신 가능하다.

```
mmp-server → "mysql:3306" 으로 DB 접근 (컨테이너 이름이 DNS 역할)
mmp-server → "mediamtx:9997" 으로 API 접근
```

---

## mmp-network 구성

```yaml
networks:
  mmp-network:
    driver: bridge

services:
  mysql:
    networks:
      - mmp-network

  mediamtx:
    networks:
      - mmp-network

  mmp-server:
    networks:
      - mmp-network

  react-app:
    networks:
      - mmp-network
```

모든 서비스가 같은 `mmp-network`에 속해 서비스 이름으로 상호 통신한다.

---

## 포트 노출 기준

### 외부에 노출해야 하는 포트 (ports 설정)

```yaml
  react-app:
    ports:
      - "80:80"      # 브라우저에서 접근하는 웹 UI

  mediamtx:
    ports:
      - "8554:8554"  # RTSP (카메라 → MediaMTX 인입)
      - "8888:8888"  # HLS (React → MediaMTX 스트리밍)
      - "8889:8889"  # WebRTC (React → MediaMTX 스트리밍)
```

**기준**: 브라우저, 카메라 장비, 외부 클라이언트가 직접 접근해야 하는 포트.

### 내부에서만 사용하는 포트 (ports 설정 없음)

```yaml
  mmp-server:
    # ports 없음 → 외부에서 직접 접근 불가
    # react-app의 nginx가 프록시로 중계

  mysql:
    # ports 없음 → 컨테이너 내부에서만 접근
    # 운영 환경에서 DB 포트를 외부에 열면 보안 위험

  mediamtx:
    # ports:
    #   - "9997:9997"  ← API 포트는 내부 전용, 외부 노출 금지
```

**기준**: 컨테이너끼리만 통신하면 되고, 외부에서 직접 접근할 이유가 없는 서비스.

---

## React → NestJS 통신: nginx 리버스 프록시

```nginx
# nginx.conf (react-app 컨테이너 내부)
location /api {
    proxy_pass http://mmp-server:3000;
}
```

브라우저는 `80`번 포트로 react-app에 접근하고, nginx가 `/api` 요청을 내부 mmp-server로 전달한다.
mmp-server의 3000 포트를 외부에 열 필요가 없다.

---

## 포트 노출 결정 트리

```
외부 클라이언트(브라우저/카메라/앱)가 직접 접속해야 하는가?
  ├── YES → ports에 명시
  └── NO  → ports 생략, 같은 네트워크 내 서비스 이름으로 통신
```
