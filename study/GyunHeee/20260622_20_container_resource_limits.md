# 컨테이너 리소스(CPU/메모리) 제한 설정 여부와 이유

## 설정 방법

```yaml
# docker-compose.yml
services:
  mmp-server:
    deploy:
      resources:
        limits:
          cpus: '1.0'        # 최대 1 CPU 코어
          memory: 512M       # 최대 512MB 메모리
        reservations:
          cpus: '0.25'       # 최소 보장 CPU
          memory: 256M       # 최소 보장 메모리

  mysql:
    deploy:
      resources:
        limits:
          memory: 1G

  mediamtx:
    deploy:
      resources:
        limits:
          cpus: '2.0'        # 스트리밍 처리에 더 많은 CPU 허용
          memory: 1G
```

---

## 제한을 설정해야 하는 이유

### 한 컨테이너가 전체 리소스를 독점하는 문제 방지

RTSP 스트림이 갑자기 많이 들어오거나 루프 버그가 생기면 MediaMTX나 mmp-server가 CPU를 100% 점유할 수 있다.
리소스 제한이 없으면 같은 서버의 MySQL도 응답 불가 상태가 된다.

```
제한 없을 때:
  MediaMTX 이상 → CPU 100% → MySQL 응답 불가 → 전체 서비스 장애

제한 있을 때:
  MediaMTX 이상 → CPU 2코어 내에서 처리 → MySQL은 정상 동작
```

---

## 설정하지 않아도 되는 경우

- 단일 서비스만 실행하는 서버 (리소스 독점 문제 없음)
- 개발 환경 (성능보다 편의성)
- 서비스 특성상 리소스 사용량이 예측 가능하고 충분한 여유가 있을 때

**학교 프로젝트 수준이라면**: 처음엔 제한 없이 개발하고, 실제 카메라를 붙여서 부하를 확인한 후 적절한 값을 설정하는 게 현실적이다. 추측으로 너무 낮게 설정하면 정상 동작하는 서비스가 OOM으로 죽는다.

---

## 메모리 제한 설정 시 주의: OOM Killer

메모리 제한을 초과하면 Docker가 컨테이너를 강제 종료한다(`OOMKilled`).

```bash
# 컨테이너가 OOM으로 죽었는지 확인
docker inspect mmp-server | grep OOMKilled
# "OOMKilled": true 이면 메모리 부족으로 종료된 것
```

제한을 설정할 때는 실제 사용량의 1.5~2배 여유를 두는 게 안전하다.

---

## 서비스별 권장 우선순위

| 서비스 | 리소스 제한 필요성 | 이유 |
|--------|------------------|------|
| MediaMTX | 높음 | 스트림 수에 따라 CPU 급증 |
| MySQL | 중간 | 메모리 캐시 사용량 크지만 예측 가능 |
| mmp-server | 중간 | 일반적으로 안정적 |
| react-app (nginx) | 낮음 | 정적 파일 서빙, 리소스 사용 적음 |
