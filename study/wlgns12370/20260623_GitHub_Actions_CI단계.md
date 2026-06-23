# 프롬프트
> GitHub Actions CI 단계에서 무슨 일이 일어나는지 설명해줘.

## CI 단계 요약

`main` 브랜치 push 시 자동 실행.

| 단계 | 내용 |
|---|---|
| 코드 체크아웃 | 최신 소스 가져오기 |
| 테스트 | 단위·통합 테스트 실행 |
| Docker 빌드 | Dockerfile로 이미지 빌드 |
| GHCR 푸시 | `ghcr.io/2026-1-2/mmp-server:latest` 태그로 푸시 |

---

## 워크플로우 예시

```yaml
jobs:
  build-and-push:
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
          tags: ghcr.io/2026-1-2/mmp-server:latest
```

---

## 실패 시 동작

CI 실패 → CD 단계 진행하지 않음 → 개발자에게 알림.
