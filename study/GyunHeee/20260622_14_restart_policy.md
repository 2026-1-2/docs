# 컨테이너 재시작 정책: unless-stopped vs always

## 정책 종류

| 정책 | 동작 |
|------|------|
| `no` | 재시작 안 함 (기본값) |
| `always` | 항상 재시작 (수동 stop 포함) |
| `unless-stopped` | 수동으로 멈춘 경우 제외하고 재시작 |
| `on-failure` | 비정상 종료(exit code != 0)일 때만 재시작 |

---

## always vs unless-stopped 차이

```
docker stop mmp-server 실행 후 Docker 데몬 재시작 시:

always        → mmp-server 자동 재시작됨
unless-stopped → 수동으로 멈췄으므로 재시작 안 됨
```

`always`는 `docker stop`으로 멈춰도 Docker 재시작 시 다시 올라온다.
`unless-stopped`는 운영자가 명시적으로 멈춘 것을 기억한다.

---

## 서비스별 권장 설정

```yaml
services:
  mysql:
    restart: unless-stopped   # DB는 항상 떠있어야 하지만, 점검 시 멈출 수 있게

  mediamtx:
    restart: unless-stopped   # 스트리밍 서버 동일

  mmp-server:
    restart: unless-stopped   # NestJS 앱 동일

  react-app:
    restart: unless-stopped   # 프론트엔드 정적 서버 동일
```

---

## 왜 always보다 unless-stopped를 선호하는가

**시나리오**: 배포 중 mmp-server를 `docker stop`으로 멈추고 설정을 변경하는 작업 중에 서버가 재부팅됨.

- `always`: Docker 재시작 후 mmp-server가 자동으로 올라와서 변경 전 상태로 동작
- `unless-stopped`: Docker 재시작 후에도 멈춘 상태 유지 → 작업 완료 후 수동으로 올리면 됨

점검, 배포, 설정 변경 등 운영자가 의도적으로 멈춘 상황에서 `unless-stopped`가 안전하다.

---

## on-failure 활용 예시

```yaml
  data-migration:
    restart: on-failure
    restart_policy:
      condition: on-failure
      max_attempts: 3     # 최대 3번만 재시도
```

일회성 마이그레이션 작업처럼 성공하면 종료되어야 하는 컨테이너에 적합하다.
