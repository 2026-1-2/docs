# 3개 레포의 CI/CD 파이프라인: 독립 vs 통합

## 레포 구성

```
video-recorder  ← 녹화 서비스 (Python/C++ 또는 별도 서비스)
mmp-server      ← NestJS 백엔드
client          ← React 프론트엔드
```

---

## 각 레포가 독립적인 CI/CD를 갖는 구조

```
[video-recorder 레포]           [mmp-server 레포]           [client 레포]
       ↓ push                        ↓ push                      ↓ push
  CI: 빌드/테스트              CI: 빌드/테스트/E2E          CI: 빌드/테스트
       ↓ main merge                  ↓ main merge                ↓ main merge
  CD: 이미지 빌드/푸쉬         CD: 이미지 빌드/푸쉬         CD: 이미지 빌드/푸쉬
       ↓                             ↓                           ↓
  운영 서버 배포               운영 서버 배포               운영 서버 배포
```

**장점**
- 각 서비스가 독립적으로 배포 가능 (client 수정이 mmp-server 배포를 트리거하지 않음)
- 레포별 개발 속도가 달라도 충돌 없음
- 한 레포 CI 실패가 다른 레포에 영향 없음

**단점**
- 세 서비스가 함께 동작해야 하는 변경 사항(API 스펙 변경 등)은 배포 순서를 수동으로 맞춰야 함

---

## 실제 워크플로우 파일 구조

```
video-recorder/
  .github/workflows/
    ci.yml       # PR에 테스트
    cd.yml       # main push 시 배포

mmp-server/
  .github/workflows/
    ci.yml
    cd.yml

client/
  .github/workflows/
    ci.yml
    cd.yml
```

---

## mmp-server CI/CD 예시

```yaml
# mmp-server/.github/workflows/cd.yml
name: CD

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: GitHub Container Registry 로그인
        run: echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: 이미지 빌드 및 푸쉬
        run: |
          docker build -t ghcr.io/${{ github.repository }}:${{ github.sha }} .
          docker build -t ghcr.io/${{ github.repository }}:latest .
          docker push ghcr.io/${{ github.repository }}:${{ github.sha }}
          docker push ghcr.io/${{ github.repository }}:latest

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    steps:
      - uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/mmp
            IMAGE_TAG=${{ github.sha }} bash run.sh mmp-server
```

---

## 서비스 간 의존성이 있는 배포 처리

API 스펙이 변경되어 mmp-server와 client를 동시에 배포해야 할 때.

### 방법 1: 배포 순서 수동 조정

```
1. mmp-server 배포 (하위 호환 API 유지)
2. client 배포 (새 API 사용)
3. mmp-server의 구 API 제거 (다음 배포 때)
```

API 변경 시 하위 호환을 유지하면 순서 의존성이 없어진다.

### 방법 2: 통합 배포 레포 (monorepo 또는 deploy 레포)

```
deploy 레포/
  docker-compose.yml   ← 전체 서비스 버전 정의
  run.sh
  .github/workflows/
    deploy.yml         ← 3개 서비스 한 번에 배포
```

```yaml
# deploy.yml
on:
  repository_dispatch:
    types: [deploy-all]  # 다른 레포에서 트리거
```

한 레포에서 전체 스택을 함께 배포할 때 사용한다.

---

## 권장 구조 결론

| 상황 | 선택 |
|------|------|
| 서비스별 배포 주기가 다름 | 독립 파이프라인 |
| API 변경이 잦고 동기화 필요 | 통합 배포 레포 추가 |
| 이 프로젝트 규모 | 독립 파이프라인 + 하위 호환 API 정책 |

세 레포가 독립 파이프라인을 갖되, API 변경 시 하위 호환을 유지하는 것이 현실적이다.
