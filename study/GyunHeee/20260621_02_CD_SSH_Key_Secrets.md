# CD 단계에서 SSH 키를 GitHub Secrets로 안전하게 관리하는 방법

## 왜 SSH 키가 필요한가

GitHub Actions 러너(GitHub 서버)에서 운영 서버(온프레미스)로 접속해서
`docker compose pull && docker compose up -d` 같은 배포 명령을 실행해야 한다.
이때 비밀번호 대신 SSH 키 인증을 사용한다.

---

## 설정 절차

### 1단계: 배포 전용 SSH 키 생성

```bash
# 로컬 또는 운영 서버에서 생성 (배포 전용 키이므로 passphrase 없이)
ssh-keygen -t ed25519 -C "github-actions-deploy" -f ~/.ssh/deploy_key -N ""

# 두 파일 생성됨:
# deploy_key      ← 개인키 (GitHub Secrets에 저장)
# deploy_key.pub  ← 공개키 (운영 서버에 등록)
```

### 2단계: 운영 서버에 공개키 등록

```bash
# 운영 서버에서
cat deploy_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

### 3단계: GitHub Secrets에 개인키 저장

GitHub 레포 → Settings → Secrets and variables → Actions → New repository secret

```
이름: SSH_PRIVATE_KEY
값:   (deploy_key 파일 내용 전체 붙여넣기)

이름: DEPLOY_HOST
값:   운영 서버 IP 또는 도메인

이름: DEPLOY_USER
값:   ubuntu (또는 배포 계정명)
```

**절대로 개인키를 코드에 커밋하지 않는다.**

---

## GitHub Actions 워크플로우에서 사용

```yaml
# .github/workflows/cd.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    needs: test    # CI 통과 후에만 실행

    steps:
      - name: SSH 키 설정
        uses: webfactory/ssh-agent@v0.9.0
        with:
          ssh-private-key: ${{ secrets.SSH_PRIVATE_KEY }}

      - name: 운영 서버 배포
        run: |
          ssh -o StrictHostKeyChecking=no \
              ${{ secrets.DEPLOY_USER }}@${{ secrets.DEPLOY_HOST }} \
              "cd /opt/mmp && bash run.sh"
```

또는 `appleboy/ssh-action`을 사용하면 더 간결하다.

```yaml
      - name: 운영 서버 배포
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /opt/mmp
            bash run.sh
```

---

## Secrets가 안전한 이유

- GitHub Secrets는 암호화되어 저장되고, 로그에 `***`로 마스킹됨
- 포크된 레포의 PR에서는 Secrets가 주입되지 않음 (외부 공격자 접근 불가)
- 개인키는 Actions 실행 중 메모리에만 존재하고 디스크에 저장되지 않음

---

## 추가 보안 조치

```bash
# 운영 서버에서 배포 계정의 권한 최소화
# sudo 없이 docker compose만 실행 가능하도록 제한
# /etc/sudoers
deploy ALL=(ALL) NOPASSWD: /usr/bin/docker compose
```

배포 계정은 배포에 필요한 명령만 실행 가능하게 제한한다.
