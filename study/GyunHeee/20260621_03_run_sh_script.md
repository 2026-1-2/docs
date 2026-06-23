# run.sh 스크립트가 수행하는 작업

## 역할

GitHub Actions가 SSH로 운영 서버에 접속한 후 실행하는 배포 자동화 스크립트다.
이미지 업데이트부터 컨테이너 재기동, 마이그레이션까지 배포에 필요한 모든 작업을 순서대로 실행한다.

---

## run.sh 전체 예시

```bash
#!/bin/bash
set -e  # 어느 단계라도 실패하면 즉시 중단

DEPLOY_DIR="/opt/mmp"
COMPOSE_FILE="$DEPLOY_DIR/docker-compose.yml"

echo "===== 배포 시작: $(date) ====="

# 1. 작업 디렉토리 이동
cd $DEPLOY_DIR

# 2. 최신 docker-compose.yml 및 설정 파일 업데이트
git pull origin main

# 3. 최신 이미지 pull
echo "[1/5] 이미지 Pull..."
docker compose pull mmp-server
docker compose pull react-app
# mediamtx, mysql은 버전 고정이므로 pull 생략 가능

# 4. DB 마이그레이션 실행 (컨테이너 기동 전)
echo "[2/5] DB 마이그레이션..."
docker compose run --rm mmp-server npm run migration:run

# 5. 컨테이너 재기동
echo "[3/5] 컨테이너 재기동..."
docker compose up -d --no-build

# 6. 헬스체크 (기동 후 정상 동작 확인)
echo "[4/5] 헬스체크..."
sleep 10  # 컨테이너 초기화 대기

HTTP_STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)
if [ "$HTTP_STATUS" != "200" ]; then
  echo "헬스체크 실패: $HTTP_STATUS"
  exit 1
fi

# 7. 오래된 이미지 정리
echo "[5/5] 미사용 이미지 정리..."
docker image prune -f

echo "===== 배포 완료: $(date) ====="
```

---

## 각 단계별 설명

### `set -e`
스크립트 어느 줄에서든 에러가 나면 즉시 중단한다.
마이그레이션 실패 후 컨테이너가 재기동되는 상황을 막는다.

### `docker compose pull`
GitHub Container Registry(ghcr.io) 또는 Docker Hub에서 새 이미지를 내려받는다.
컨테이너를 재기동하기 전에 먼저 받아두므로 재기동 시간이 줄어든다.

### `docker compose run --rm mmp-server npm run migration:run`
새 컨테이너로 마이그레이션만 실행하고 바로 종료한다.
기존 컨테이너는 이 시점에 아직 살아있다.

### `docker compose up -d --no-build`
`--no-build`: 이미지를 다시 빌드하지 않고 pull한 이미지를 사용한다.
변경된 서비스만 재기동된다 (변경 없는 서비스는 그대로).

### 헬스체크
기동 후 실제로 HTTP 응답을 받을 수 있는지 확인한다.
실패 시 `exit 1`로 스크립트 종료 → GitHub Actions에 실패 신호 전달.

---

## 환경변수 주입

```bash
# run.sh에서 .env 파일 로드
export $(cat $DEPLOY_DIR/.env | xargs)
```

또는 docker-compose.yml의 `env_file` 설정으로 자동 주입.

---

## 실행 로그 확인

```bash
# GitHub Actions 로그에서 직접 확인 가능
# 또는 운영 서버에서
tail -f /opt/mmp/deploy.log
```
