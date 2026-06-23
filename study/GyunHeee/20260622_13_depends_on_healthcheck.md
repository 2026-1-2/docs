# depends_on만으로 시작 순서가 보장되는가? healthcheck가 왜 필요한가?

## depends_on의 한계

```yaml
services:
  mmp-server:
    depends_on:
      - mysql
```

`depends_on`은 **컨테이너가 시작(start)되는 순서**만 보장한다.
MySQL 컨테이너가 실행 중(running)이라고 해서 MySQL 데몬이 실제로 요청을 받을 준비가 된 건 아니다.

```
MySQL 컨테이너 시작
      ↓ (depends_on이 여기까지만 기다림)
MySQL 초기화 중... (아직 연결 불가)
      ↓
MySQL 준비 완료 (연결 가능)
```

mmp-server가 MySQL이 초기화되는 동안 연결을 시도하면 `ECONNREFUSED`로 죽는다.

---

## healthcheck + condition으로 해결

```yaml
services:
  mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p$$MYSQL_ROOT_PASSWORD"]
      interval: 10s      # 10초마다 체크
      timeout: 5s        # 응답 대기 최대 5초
      retries: 5         # 5번 실패하면 unhealthy
      start_period: 30s  # 시작 후 30초간은 실패해도 unhealthy 처리 안 함

  mmp-server:
    depends_on:
      mysql:
        condition: service_healthy  # healthy 상태가 될 때까지 대기
```

`condition: service_healthy`를 쓰면 MySQL의 healthcheck가 통과될 때까지 mmp-server 시작을 미룬다.

---

## MediaMTX도 동일하게 적용

```yaml
  mediamtx:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9997/v3/paths/list"]
      interval: 10s
      timeout: 3s
      retries: 3

  mmp-server:
    depends_on:
      mysql:
        condition: service_healthy
      mediamtx:
        condition: service_healthy
```

---

## 요약

| | depends_on (기본) | depends_on + healthcheck |
|--|------------------|------------------------|
| 보장하는 것 | 컨테이너 시작 순서 | 서비스 실제 준비 상태 |
| MySQL 초기화 대기 | 불가 | 가능 |
| 실제 운영 적합성 | 낮음 | 높음 |
