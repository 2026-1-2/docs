# 멀티 컨테이너 구조에서 로그 수집·확인 방법

## 기본: docker logs

```bash
# 특정 서비스 로그 실시간 확인
docker compose logs -f mmp-server

# 모든 서비스 로그 동시 확인
docker compose logs -f

# 마지막 100줄만
docker compose logs --tail=100 mmp-server

# 특정 시간 이후 로그
docker compose logs --since="2026-06-22T10:00:00" mmp-server
```

컨테이너가 stdout/stderr에 출력하는 내용이 모두 여기 잡힌다.
NestJS의 `Logger`, MediaMTX의 내부 로그 모두 포함.

---

## NestJS 로그 설정

```typescript
// main.ts
const app = await NestFactory.create(AppModule, {
  logger: ['log', 'error', 'warn', 'debug'],
});
```

```typescript
// 서비스에서 사용
import { Logger } from '@nestjs/common';

@Injectable()
export class CameraService {
  private readonly logger = new Logger(CameraService.name);

  async create(dto: CreateCameraDto) {
    this.logger.log(`카메라 등록 시작: ${dto.name}`);
    // ...
    this.logger.error(`MediaMTX 등록 실패: ${error.message}`);
  }
}
```

---

## Docker 로그 드라이버 설정

기본값은 `json-file`로 로컬 디스크에 쌓인다. 용량 제한을 설정하지 않으면 디스크를 가득 채울 수 있다.

```yaml
# docker-compose.yml
services:
  mmp-server:
    logging:
      driver: json-file
      options:
        max-size: "50m"   # 파일당 최대 50MB
        max-file: "5"     # 최대 5개 파일 (총 250MB)
```

---

## 로그 파일 위치

```bash
# 컨테이너 로그 파일 위치 확인
docker inspect mmp-server | grep LogPath

# 보통 이 경로에 있음
/var/lib/docker/containers/<container-id>/<container-id>-json.log
```

---

## 운영 환경에서 별도 로깅 솔루션

온프레미스 소규모 프로젝트라면 `docker logs`로 충분하다.
규모가 커지면 중앙화된 로그 수집을 고려한다.

### 경량 옵션: Loki + Grafana
```yaml
# docker-compose.yml에 추가
  loki:
    image: grafana/loki:latest

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
```

Docker 로그 드라이버를 loki로 변경하면 모든 컨테이너 로그가 Loki로 수집되고, Grafana에서 검색·시각화 가능.

### 에러 알람 연동
심각한 에러는 NestJS에서 직접 SSE나 알람 테이블에 기록하는 것으로 충분한 경우가 많다.
별도 APM(Application Performance Monitoring) 솔루션은 필요할 때 추가한다.

---

## 실용적 운영 팁

```bash
# 에러만 필터링
docker compose logs mmp-server 2>&1 | grep ERROR

# 특정 카메라 관련 로그
docker compose logs mmp-server | grep "cam1"

# 로그를 파일로 저장
docker compose logs --no-color mmp-server > server.log
```
