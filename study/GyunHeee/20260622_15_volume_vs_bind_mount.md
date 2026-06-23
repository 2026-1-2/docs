# Docker Volume vs Bind Mount: 녹화 파일 저장에 무엇을 쓰는가

## 차이점

| | Docker Volume | Bind Mount |
|--|--------------|------------|
| 경로 관리 | Docker가 관리 (`/var/lib/docker/volumes/`) | 호스트 경로를 직접 지정 |
| 호스트 접근 | 어렵 (Docker 명령어로 접근) | 쉬움 (그냥 파일 탐색기/shell) |
| 이식성 | 높음 (경로 독립적) | 낮음 (호스트 경로에 의존) |
| 성능 | 동등하거나 약간 빠름 | 동등 |
| 용도 | DB 데이터, 설정 파일 | 개발 소스 코드, 외부 HDD 마운트 |

---

## 녹화 파일(HDD) 저장에는 Bind Mount가 적합하다

```yaml
services:
  mediamtx:
    volumes:
      - /mnt/hdd1/recordings:/recordings  # bind mount
```

**이유**

1. **외부 HDD 직접 연결**: 온프레미스 서버에서 HDD를 `/mnt/hdd1`에 마운트하면, 해당 경로를 그대로 컨테이너에 연결해야 한다. Docker Volume은 이 경로를 추상화할 수 없다.

2. **운영 중 파일 직접 접근 필요**: 녹화 파일을 운영자가 직접 확인하거나 백업할 때 Docker 명령어 없이 파일 시스템에서 바로 접근해야 한다.

3. **디스크 용량 모니터링**: HDD 용량을 `df -h /mnt/hdd1`로 바로 확인 가능.

---

## DB 데이터는 Docker Volume이 적합하다

```yaml
services:
  mysql:
    volumes:
      - mysql-data:/var/lib/mysql  # named volume

volumes:
  mysql-data:
```

MySQL 데이터는 Docker가 관리하는 게 안전하다. 실수로 호스트에서 파일을 건드릴 위험이 없다.

---

## 실제 구성 예시

```yaml
services:
  mediamtx:
    volumes:
      - /mnt/hdd1/recordings:/recordings       # bind mount (외부 HDD)
      - ./mediamtx.yml:/mediamtx.yml:ro        # bind mount (설정 파일, 읽기 전용)

  mmp-server:
    volumes:
      - /mnt/hdd1/snapshots:/snapshots         # bind mount (스냅샷)
      - /mnt/hdd1/recordings:/recordings:ro    # bind mount (읽기 전용, 파일 경로 참조용)

  mysql:
    volumes:
      - mysql-data:/var/lib/mysql              # named volume (DB 데이터)

volumes:
  mysql-data:
```

---

## 핵심 기준

- **HDD에 물리적으로 저장해야 하는 파일** → Bind Mount
- **Docker가 관리하면 되는 데이터** → Named Volume
