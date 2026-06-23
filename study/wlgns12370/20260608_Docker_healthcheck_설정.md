# 프롬프트
> Docker healthcheck를 설정하는 방법과 MMP에서 어떻게 활용하는지 설명해줘.

## healthcheck란

컨테이너가 실행 중인지뿐 아니라 **실제로 정상 동작 중인지** 주기적으로 확인하는 기능.

---

## 설정 옵션

```yaml
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  interval: 10s    # 검사 주기
  timeout: 5s      # 단일 검사 타임아웃
  retries: 5       # 실패 허용 횟수
  start_period: 30s  # 초기 대기 시간
```

---

## 상태 값

| 상태 | 의미 |
|---|---|
| `starting` | start_period 이내 (아직 판정 안 함) |
| `healthy` | 마지막 검사 성공 |
| `unhealthy` | retries 횟수 초과 실패 |

---

## MMP 활용

```yaml
mmp-api:
  depends_on:
    mmp-mysql:
      condition: service_healthy  # MySQL이 healthy 상태일 때만 시작
```

MySQL healthcheck 미설정 시 → mmp-api가 DB 준비 전에 시작 → 연결 오류 → 재시작 루프 발생.

---

## 서비스별 healthcheck 예시

```yaml
# MySQL
test: ["CMD", "mysqladmin", "ping"]

# HTTP API
test: ["CMD", "curl", "-f", "http://localhost:3000/health"]

# MediaMTX
test: ["CMD", "curl", "-f", "http://localhost:9997/v3/paths/list"]
```
