# 멀티스테이지 빌드(Multi-stage Build)를 NestJS/React에 적용하기

## 왜 멀티스테이지 빌드를 쓰는가

```
단일 스테이지 빌드 이미지 크기: ~1.2GB (node_modules, 빌드 도구 포함)
멀티스테이지 빌드 이미지 크기: ~200MB (실행에 필요한 것만)
```

빌드에 필요한 도구(TypeScript 컴파일러, 개발 의존성)는 최종 이미지에 포함될 필요가 없다.
멀티스테이지 빌드는 빌드용 이미지와 실행용 이미지를 분리한다.

---

## NestJS Dockerfile

```dockerfile
# 1단계: 빌드
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci                    # devDependencies 포함 설치

COPY . .
RUN npm run build             # TypeScript → JavaScript 컴파일


# 2단계: 실행
FROM node:20-alpine AS runner
WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev         # production 의존성만 설치

COPY --from=builder /app/dist ./dist   # 빌드 결과물만 복사

ENV NODE_ENV=production
EXPOSE 3000
CMD ["node", "dist/main.js"]
```

`--from=builder`로 빌드 스테이지의 결과물만 가져온다.
TypeScript 소스, node_modules의 개발 도구 등은 최종 이미지에 포함되지 않는다.

---

## React Dockerfile

```dockerfile
# 1단계: 빌드
FROM node:20-alpine AS builder
WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build             # React → 정적 파일(HTML/JS/CSS) 생성


# 2단계: Nginx로 정적 파일 서빙
FROM nginx:alpine AS runner

COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

React 빌드 결과는 정적 파일이므로 Node.js 없이 nginx만으로 서빙한다.
최종 이미지에 Node.js가 전혀 없다.

---

## nginx.conf (React + API 프록시)

```nginx
server {
    listen 80;

    location / {
        root /usr/share/nginx/html;
        try_files $uri $uri/ /index.html;  # SPA 라우팅
    }

    location /api {
        proxy_pass http://mmp-server:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## docker-compose.yml 빌드 설정

```yaml
services:
  mmp-server:
    build:
      context: ./backend
      dockerfile: Dockerfile
      target: runner          # 특정 스테이지 지정 (생략 시 마지막 스테이지)

  react-app:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: runner
```

---

## 이미지 크기 비교 예시

```bash
docker images

REPOSITORY    SIZE
mmp-server    185MB    # 멀티스테이지
react-app      28MB    # nginx + 정적 파일만
```

빌드 시간은 초기에 조금 더 걸리지만, 이미지 크기와 보안 공격 면(surface)이 줄어든다.
