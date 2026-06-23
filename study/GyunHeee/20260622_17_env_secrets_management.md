# 환경변수와 시크릿을 Docker Compose에서 안전하게 관리하는 방법

## 기본: .env 파일 분리

```
.env                 ← gitignore에 추가 (실제 값)
.env.example         ← git에 커밋 (키 이름만, 값 없음)
```

```env
# .env.example (git에 올라가는 파일)
DB_PASSWORD=
JWT_SECRET=
WEBHOOK_SECRET=
MEDIAMTX_API_URL=
```

```env
# .env (로컬/서버에만 존재, gitignore)
DB_PASSWORD=실제비밀번호
JWT_SECRET=실제시크릿32자이상
WEBHOOK_SECRET=랜덤값
MEDIAMTX_API_URL=http://mediamtx:9997
```

---

## Docker Compose에서 .env 사용

```yaml
# docker-compose.yml
services:
  mysql:
    environment:
      MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}
      MYSQL_DATABASE: ${DB_DATABASE}

  mmp-server:
    env_file:
      - .env   # 파일 통째로 주입
    # 또는 개별 지정
    environment:
      DB_PASSWORD: ${DB_PASSWORD}
      JWT_SECRET: ${JWT_SECRET}
```

`env_file`은 파일의 모든 키-값을 컨테이너에 주입한다.
`environment`는 필요한 값만 선택해서 주입한다.

---

## 시크릿 값을 docker-compose.yml에 직접 쓰면 안 되는 이유

```yaml
# 나쁜 예: 절대 하지 말 것
services:
  mysql:
    environment:
      MYSQL_ROOT_PASSWORD: my_actual_password  # git에 올라가면 끝
```

`docker-compose.yml`은 git에 올라간다. 시크릿 값이 들어가면 히스토리에 영구 기록된다.

---

## 운영 환경에서의 추가 보안 조치

### 방법 1: Docker Secrets (Swarm 모드)
```yaml
services:
  mysql:
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt  # 파일로 주입, 메모리에서만 읽힘
```

### 방법 2: 서버 환경변수로 주입
CI/CD 파이프라인(GitHub Actions, Jenkins)에서 시크릿을 서버 환경변수로 설정하고,
`docker compose up` 실행 시 자동으로 `.env` 없이 주입.

```bash
# 서버 shell에서 직접 설정
export DB_PASSWORD="실제비밀번호"
docker compose up -d
```

---

## .gitignore 설정

```gitignore
# .gitignore
.env
.env.local
.env.production
secrets/
```

---

## 체크리스트

- [ ] `.env` 파일이 `.gitignore`에 있는가
- [ ] `docker-compose.yml`에 하드코딩된 시크릿이 없는가
- [ ] `.env.example`에 키 이름만 있고 실제 값은 없는가
- [ ] git 히스토리에 시크릿이 들어간 커밋이 없는가
