# Docker Compose 기반 멀티 컨테이너 인프라 구축

## 1. 개요

버티포트 CCTV 감시 시스템은 여러 서비스(미디어 서버, 백엔드 API, DB, 녹화 서버)가 동시에 동작해야 한다.
각 서비스를 독립된 컨테이너로 분리하고 Docker Compose로 오케스트레이션하는 구조를 설계한다.

---

## 2. Docker vs Docker Compose

| 항목 | Docker 단독 | Docker Compose |
|------|-------------|----------------|
| 컨테이너 관리 | `docker run` 개별 실행 | `docker-compose up` 일괄 실행 |
| 네트워크 | 수동으로 네트워크 생성/연결 | 자동으로 서비스 간 네트워크 구성 |
| 환경 변수 | `-e` 플래그로 개별 전달 | `.env` 파일로 중앙 관리 |
| 의존성 관리 | 불가능 | `depends_on`으로 시작 순서 제어 |
| 볼륨 | `-v` 플래그로 개별 마운트 | `volumes` 섹션에서 선언적 관리 |

---

## 3. 프로젝트 컨테이너 구성

```
docker-compose.yml
├── mediamtx        (미디어 서버 - RTSP/WebRTC 변환)
├── mmp-server      (NestJS 백엔드 API)
├── mysql           (데이터베이스)
├── video-recorder  (Python 영상 녹화)
└── client          (React 프론트엔드 - 개발 시)
```

---

## 4. docker-compose.yml 기본 구조

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
      MYSQL_DATABASE: ${DB_NAME}
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  mediamtx:
    image: bluenviron/mediamtx:latest
    ports:
      - "8554:8554"
      - "8889:8889"
      - "8189:8189/udp"
    volumes:
      - ./mediamtx.yml:/mediamtx.yml

  mmp-server:
    build: ./mmp-server
    ports:
      - "3000:3000"
    depends_on:
      mysql:
        condition: service_healthy
    environment:
      DATABASE_URL: mysql://root:${DB_ROOT_PASSWORD}@mysql:3306/${DB_NAME}

volumes:
  mysql_data:
```

---

## 5. 핵심 개념 정리

### 5-1. 네트워크 격리

Docker Compose는 기본적으로 프로젝트 이름 기반의 브리지 네트워크를 자동 생성한다.
같은 `docker-compose.yml`에 정의된 서비스끼리는 서비스 이름으로 통신이 가능하다.

```
mmp-server → mysql:3306      (서비스 이름으로 접근)
mmp-server → mediamtx:9997   (API 호출)
```

### 5-2. 볼륨 마운트

```yaml
volumes:
  - mysql_data:/var/lib/mysql        # Named Volume (데이터 영속성)
  - ./mediamtx.yml:/mediamtx.yml     # Bind Mount (설정 파일)
  - ./recordings:/app/recordings     # Bind Mount (녹화 파일)
```

- **Named Volume**: Docker가 관리하는 영속 스토리지. DB 데이터에 적합.
- **Bind Mount**: 호스트 파일시스템의 특정 경로를 컨테이너에 연결. 설정 파일이나 녹화 파일에 적합.

### 5-3. healthcheck와 depends_on

`depends_on`만으로는 서비스가 "준비 완료"되었는지 보장하지 않는다.
MySQL 컨테이너가 시작되더라도 실제 DB가 쿼리를 받을 준비가 되기까지 수 초가 걸린다.

```yaml
depends_on:
  mysql:
    condition: service_healthy   # healthcheck 통과 후 시작
```

### 5-4. 환경 변수 관리

```bash
# .env 파일
DB_ROOT_PASSWORD=securepassword
DB_NAME=mmp_db
JWT_SECRET=my-jwt-secret
```

`docker-compose.yml`에서 `${변수명}` 형태로 참조한다.
`.env` 파일은 반드시 `.gitignore`에 추가하고, `.env.example`을 커밋한다.

---

## 6. 유용한 명령어

```bash
docker-compose up -d              # 백그라운드 실행
docker-compose down               # 중지 및 컨테이너 삭제
docker-compose logs -f mediamtx   # 특정 서비스 로그 실시간 확인
docker-compose ps                 # 서비스 상태 확인
docker-compose exec mysql bash    # 컨테이너 내부 접속
docker-compose build --no-cache   # 이미지 재빌드
```

---

## 7. 프로젝트 적용 시 고려사항

- 개발 환경과 운영 환경의 `docker-compose.yml`을 분리하는 것이 좋다 (`docker-compose.dev.yml`, `docker-compose.prod.yml`)
- MediaMTX의 UDP 포트(`8189`)는 반드시 `/udp`를 명시해야 WebRTC가 정상 동작한다
- MySQL 데이터는 Named Volume으로 관리하여 `docker-compose down` 시에도 데이터가 보존되도록 한다
- 컨테이너 간 통신은 호스트 포트가 아닌 내부 네트워크(서비스 이름)를 사용해야 한다
