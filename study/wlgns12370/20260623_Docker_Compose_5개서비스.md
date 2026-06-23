# 프롬프트
> MMP Docker Compose 구성을 설명해줘.

## docker-compose.yml 핵심 구조

```yaml
services:
  mmp-mysql:
    image: mysql:8.0
    healthcheck:
      test: ["CMD", "mysqladmin", "ping"]

  mmp-api:
    image: ghcr.io/2026-1-2/mmp-server:latest
    depends_on:
      mmp-mysql:
        condition: service_healthy

  mmp-client:
    image: ghcr.io/2026-1-2/client:latest
    ports:
      - "8002:80"

  mmp-mediamtx:
    image: bluenviron/mediamtx:latest
    ports:
      - "8554:8554"
      - "8189:8189/udp"
      - "8189:8189/tcp"

  mmp-recorder:
    image: video-recorder:latest

networks:
  mmp-net:
    driver: bridge
```

---

## 단일 명령 실행

```bash
docker compose up -d    # 전체 스택 백그라운드 실행
docker compose ps       # 상태 확인
docker compose logs -f  # 로그 모니터링
```
