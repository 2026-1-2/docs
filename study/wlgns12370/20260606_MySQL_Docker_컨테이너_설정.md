# 프롬프트
> MMP에서 MySQL을 Docker 컨테이너로 운영하는 설정을 설명해줘.

## MySQL 컨테이너 설정

```yaml
mmp-mysql:
  image: mysql:8.0
  environment:
    MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
    MYSQL_DATABASE: mmp
  volumes:
    - mysql_data:/var/lib/mysql      # HDD 마운트
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
    interval: 10s
    timeout: 5s
    retries: 5
  networks:
    - mmp-net
```

---

## Healthcheck가 중요한 이유

MySQL은 컨테이너가 시작되어도 **초기화가 완료될 때까지 수십 초 소요**.  
mmp-api가 MySQL 준비 전에 시작하면 연결 실패 → 서비스 크래시.

```yaml
mmp-api:
  depends_on:
    mmp-mysql:
      condition: service_healthy  # healthcheck 통과 후 시작
```

---

## 데이터 저장 경로

```
컨테이너 내부: /var/lib/mysql
호스트 HDD:    /hdd/mmp-data/mysql
```

컨테이너 삭제 후에도 HDD 데이터 유지 → 재시작 시 데이터 복구.
