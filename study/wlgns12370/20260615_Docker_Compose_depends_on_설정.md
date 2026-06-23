# 프롬프트
> Docker Compose에서 depends_on을 설정하는 방법과 주의사항을 설명해줘.

## 기본 depends_on (단순 순서만 보장)

```yaml
mmp-api:
  depends_on:
    - mmp-mysql  # MySQL 컨테이너 시작 후 mmp-api 시작
                  # 단, MySQL이 준비됐는지는 확인 안 함
```

문제: MySQL 컨테이너가 시작되었어도 DB 초기화 완료 전일 수 있음.

---

## condition 기반 depends_on (권장)

```yaml
mmp-api:
  depends_on:
    mmp-mysql:
      condition: service_healthy  # healthcheck 통과 후 시작
```

`service_healthy` — MySQL healthcheck가 성공할 때까지 mmp-api 시작 대기.

---

## condition 종류

| condition | 의미 |
|---|---|
| `service_started` | 컨테이너 시작됨 (기본값) |
| `service_healthy` | healthcheck 통과 |
| `service_completed_successfully` | 종료 코드 0으로 완료 |

---

## MMP 설정

```yaml
mmp-api:
  depends_on:
    mmp-mysql:
      condition: service_healthy

mmp-mysql:
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    retries: 5
```

MySQL이 완전히 준비된 후 mmp-api 시작 → 연결 오류 방지.
