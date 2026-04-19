# GitHub Actions CI/CD 파이프라인 구축

## 1. CI/CD 개요

CI(Continuous Integration)는 코드 변경 사항을 자동으로 빌드/테스트하는 것이고,
CD(Continuous Deployment)는 테스트를 통과한 코드를 자동으로 배포하는 것이다.

GitHub Actions는 GitHub에 내장된 CI/CD 플랫폼으로, `.github/workflows/` 디렉토리에 YAML 파일을 작성하여 파이프라인을 정의한다.

---

## 2. GitHub Actions 기본 구조

```yaml
name: CI Pipeline             # 워크플로우 이름

on:                            # 트리거 조건
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:                          # 작업 정의
  build:
    runs-on: ubuntu-latest     # 실행 환경
    steps:
      - uses: actions/checkout@v4
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run build
```

---

## 3. 프로젝트 CI/CD 파이프라인 설계

### 3-1. mmp-server (NestJS 백엔드)

```yaml
name: MMP Server CI/CD

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: test
          MYSQL_DATABASE: mmp_test
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npx prisma migrate deploy
        env:
          DATABASE_URL: mysql://root:test@localhost:3306/mmp_test
      - run: npm run test

  docker:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

### 3-2. client (React 프론트엔드)

```yaml
name: Client CI

on:
  pull_request:
    branches: [main, dev]

jobs:
  lint-and-build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
      - run: npm ci
      - run: npm run lint
      - run: npm run build
```

---

## 4. Docker 이미지 빌드 및 배포

### 4-1. GitHub Container Registry (GHCR)

GitHub Actions에서 빌드한 Docker 이미지를 GHCR에 푸시하면,
운영 서버에서 `docker pull`로 최신 이미지를 가져올 수 있다.

```
개발자 → Push → GitHub Actions → Docker Build → GHCR → 운영 서버 Pull
```

### 4-2. Dockerfile (NestJS)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npx prisma generate
RUN npm run build

FROM node:20-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/prisma ./prisma
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

멀티 스테이지 빌드를 사용하여 최종 이미지 크기를 최소화한다.

---

## 5. Secrets 관리

민감한 정보(DB 비밀번호, JWT 시크릿 등)는 GitHub Secrets에 저장한다.

```
Repository Settings → Secrets and variables → Actions
```

워크플로우에서 참조:
```yaml
env:
  DATABASE_URL: ${{ secrets.DATABASE_URL }}
  JWT_SECRET: ${{ secrets.JWT_SECRET }}
```

`.env` 파일은 절대 커밋하지 않으며, `.env.example`만 커밋한다.

---

## 6. 브랜치 전략과 CI/CD 연동

```
main     ─── 운영 배포 (push 시 Docker 빌드 + 배포)
  │
dev      ─── 개발 통합 (PR 시 테스트)
  │
feat/#1  ─── 기능 개발 (PR → dev 병합)
```

| 이벤트 | 트리거 | 수행 작업 |
|--------|--------|-----------|
| feat → dev PR | pull_request | lint, build, test |
| dev → main PR | pull_request | lint, build, test |
| main push | push | Docker 빌드 + GHCR 배포 |

---

## 7. 프로젝트 적용 현황

- `mmp-server`에 `.github/workflows/docker-publish.yml` 추가 완료
- main 브랜치 push 시 자동으로 Docker 이미지 빌드 및 GHCR 배포
- PR 생성 시 자동 빌드 검증으로 코드 품질 유지
