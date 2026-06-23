# 배포 실패 시 롤백 방법

## 롤백의 두 가지 차원

배포 실패는 두 가지로 나뉜다.
1. **이미지/컨테이너 문제** → 이전 Docker 이미지로 롤백
2. **DB 마이그레이션 문제** → DB 스키마 롤백 (더 어렵고 위험)

---

## 방법 1: Docker 이미지 태그로 롤백

이미지를 `latest`만 쓰면 이전 버전으로 돌아갈 수 없다.
태그를 사용하면 특정 버전으로 롤백이 가능하다.

```yaml
# docker-compose.yml
services:
  mmp-server:
    image: ghcr.io/org/mmp-server:${IMAGE_TAG:-latest}
```

```bash
# 배포 시 커밋 SHA를 태그로 사용
IMAGE_TAG=$GITHUB_SHA docker compose up -d

# 롤백 시 이전 태그로 교체
IMAGE_TAG=abc1234 docker compose up -d
```

GitHub Actions에서 자동으로 태그 관리:

```yaml
- name: 이미지 빌드 및 푸쉬
  run: |
    docker build -t ghcr.io/org/mmp-server:${{ github.sha }} .
    docker build -t ghcr.io/org/mmp-server:latest .
    docker push ghcr.io/org/mmp-server:${{ github.sha }}
    docker push ghcr.io/org/mmp-server:latest
```

---

## 방법 2: run.sh에 자동 롤백 내장

```bash
#!/bin/bash
set -e

# 현재 실행 중인 이미지 태그 저장
CURRENT_TAG=$(docker inspect mmp-server --format='{{.Config.Image}}')

deploy() {
  docker compose pull mmp-server
  docker compose run --rm mmp-server npm run migration:run
  docker compose up -d mmp-server
}

rollback() {
  echo "배포 실패, 롤백 시작..."
  # 이전 이미지로 복원
  docker tag $CURRENT_TAG ghcr.io/org/mmp-server:rollback
  IMAGE_TAG=rollback docker compose up -d mmp-server
  echo "롤백 완료"
}

# 배포 시도, 실패 시 롤백
if deploy; then
  # 헬스체크
  sleep 10
  if ! curl -sf http://localhost:3000/health; then
    rollback
    exit 1
  fi
  echo "배포 성공"
else
  rollback
  exit 1
fi
```

---

## DB 마이그레이션 롤백

마이그레이션은 롤백이 어렵기 때문에 예방이 중요하다.

### 예방: 하위 호환 마이그레이션 작성

```sql
-- 나쁜 예: 컬럼 삭제 (롤백 불가)
ALTER TABLE cameras DROP COLUMN old_field;

-- 좋은 예: 새 컬럼 추가 (롤백 가능)
ALTER TABLE cameras ADD COLUMN new_field VARCHAR(255);
-- 데이터 마이그레이션 후 old_field는 나중에 삭제
```

### TypeORM revert 명령

```bash
# 마지막 마이그레이션 되돌리기
docker compose run --rm mmp-server npm run migration:revert
```

---

## GitHub Actions에서 롤백 워크플로우

```yaml
# .github/workflows/rollback.yml
name: Rollback

on:
  workflow_dispatch:
    inputs:
      image_tag:
        description: '롤백할 이미지 태그 (커밋 SHA)'
        required: true

jobs:
  rollback:
    runs-on: ubuntu-latest
    steps:
      - uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/mmp
            IMAGE_TAG=${{ github.event.inputs.image_tag }} docker compose up -d mmp-server
```

수동으로 실행(workflow_dispatch)해서 특정 커밋 SHA로 롤백한다.

---

## 핵심 원칙

- 이미지는 항상 커밋 SHA로 태그 → 언제든 특정 버전으로 복원 가능
- 마이그레이션은 하위 호환으로 작성 → 코드 롤백 후 DB 롤백 없이도 동작
- 헬스체크를 배포 스크립트에 포함 → 실패를 즉시 감지
