# 프롬프트
> GitHub Actions 워크플로우가 언제, 어떻게 트리거되는지 설명해줘.

## 트리거 종류

```yaml
on:
  push:
    branches: [main]          # main 브랜치 push 시
  pull_request:
    branches: [main]          # PR 생성/업데이트 시
  workflow_dispatch:           # 수동 실행
  schedule:
    - cron: '0 9 * * 1'       # 매주 월요일 오전 9시
```

---

## MMP CI/CD 트리거

```yaml
on:
  push:
    branches: [main]
```

`main` 브랜치 push 시에만 전체 CI/CD 파이프라인 실행.  
PR 단계에서는 테스트만 실행, 배포는 main 머지 후 진행.

---

## 브랜치 전략

```
feature/* 브랜치
    → PR 생성 (테스트만 실행)
    → 코드 리뷰
    → main 머지
    → CI/CD 자동 배포
```

---

## 조건부 실행

```yaml
jobs:
  deploy:
    if: github.ref == 'refs/heads/main'  # main에서만 배포
    needs: test                           # test 잡 성공 후 실행
```
