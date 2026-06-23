# 무중단 배포 전략 vs 단순 재시작

## 단순 재시작 방식 (현실적 선택)

```bash
docker compose up -d
```

`docker compose up -d`는 변경된 서비스만 재기동한다.
컨테이너가 멈추고 다시 뜨는 동안 수 초~수십 초의 다운타임이 발생한다.

**이 방식을 선택하는 이유**
- 구현이 단순하다
- 온프레미스 단일 서버에서는 롤링 업데이트 인프라가 없다
- 새벽 유지보수 시간에 배포하면 다운타임이 문제되지 않는다
- 카메라 모니터링 시스템은 짧은 다운타임이 치명적이지 않다

---

## 무중단 배포를 위한 전략들

### 전략 1: Blue-Green 배포

```
현재: Blue(포트 3000) → Caddy → 사용자
배포: Green(포트 3001) 기동 및 검증
전환: Caddy가 Green으로 트래픽 전환
종료: Blue 컨테이너 종료
```

```bash
# blue-green 배포 스크립트 예시
# 1. Green 기동
docker compose -f docker-compose.green.yml up -d

# 2. 헬스체크
sleep 10
curl -f http://localhost:3001/health

# 3. Caddy 설정 변경 (트래픽 전환)
sed -i 's/mmp-server:3000/mmp-server-green:3001/' /etc/caddy/Caddyfile
docker exec caddy caddy reload --config /etc/caddy/Caddyfile

# 4. Blue 종료
docker compose -f docker-compose.blue.yml down
```

**단점**: 서버 리소스가 2배 필요. 단일 서버에서 부담이 크다.

### 전략 2: Caddy의 upstream 교체

```caddyfile
mmp-server.com {
    reverse_proxy {
        to mmp-server:3000
        lb_policy round_robin
        health_uri /health
        health_interval 10s
    }
}
```

헬스체크가 실패한 upstream을 Caddy가 자동으로 제외한다.
복수의 인스턴스가 있을 때 유효하다. 단일 인스턴스에서는 효과 없다.

---

## 이 프로젝트의 현실적 선택

```bash
# run.sh에서 다운타임을 최소화하는 방식
docker compose pull          # 미리 이미지 받기 (시간이 걸리는 작업을 컨테이너 살아있는 동안 처리)
docker compose run --rm migration  # 마이그레이션 먼저
docker compose up -d         # 재기동 (실제 다운타임은 여기서만 발생)
```

이미지를 미리 pull해두면 `up -d` 시 이미지 다운로드 없이 바로 기동된다.
실제 다운타임은 컨테이너 재시작 시간(수 초)으로 최소화된다.

---

## 다운타임이 진짜 문제가 되는 시점

서비스가 24/7 무중단 운영을 요구하고 사용자가 많아지면:
1. Kubernetes (K8s) 롤링 업데이트 도입
2. 로드밸런서 뒤에 복수 인스턴스 운영

현재 프로젝트 규모에서는 단순 재시작이 적합하다.
