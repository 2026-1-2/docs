# 프롬프트
> React SPA를 nginx로 배포하는 패턴을 설명해줘. MMP mmp-client 컨테이너 기준으로.

## React SPA 빌드

```bash
npm run build
# → /dist 또는 /build 디렉터리에 정적 파일 생성
```

---

## Dockerfile 구조

```dockerfile
# Stage 1: 빌드
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: nginx 서빙
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

멀티스테이지 빌드 → 최종 이미지에 node_modules 포함 안 됨 (이미지 경량화).

---

## SPA용 nginx.conf

```nginx
location / {
    root /usr/share/nginx/html;
    index index.html;
    try_files $uri $uri/ /index.html;  # SPA 라우팅 처리
}
```

`try_files ... /index.html` — React Router 사용 시 새로고침에도 404 방지.

---

## API 프록시

```nginx
location /api/ {
    proxy_pass http://mmp-api:3000/;
}
```

클라이언트에서 `/api/cameras` 요청 → nginx가 `mmp-api:3000/cameras`로 포워딩.
