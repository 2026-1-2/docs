# 프롬프트
> GitHub Actions CD 단계에서 운영 서버에 어떻게 자동 배포되는지 설명해줘.

## CD 단계

CI 성공 후 자동으로 운영 서버에 배포.

```yaml
deploy:
  needs: build-and-push
  steps:
    - uses: appleboy/ssh-action@v1
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        key: ${{ secrets.SSH_PRIVATE_KEY }}
        script: |
          cd /home/ubuntu/mmp-server/infra
          bash run.sh
```

---

## run.sh 내용

```bash
docker compose pull     # GHCR에서 최신 이미지 pull
docker compose up -d    # 컨테이너 재시작
docker system prune -f  # 이전 이미지 정리
```

---

## GitHub Secrets 설정

| 시크릿 | 값 |
|---|---|
| `SERVER_HOST` | 운영 서버 IP |
| `SERVER_USER` | SSH 사용자명 |
| `SSH_PRIVATE_KEY` | SSH 개인키 |
