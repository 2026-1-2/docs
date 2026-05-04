# 브랜치 / Issue / PR 컨벤션 초안

## 목적

- FE, BE, 문서 작업 과정에서 팀원 간 협업 기준을 맞추기 위해 브랜치, Issue, PR 규칙을 정리한다.
- 작업의 배경, 범위, 완료 조건을 Issue로 남기고, 실제 변경은 브랜치에서 분리하여 관리한다.
- PR을 통해 변경 이유, 테스트 결과, 리뷰 포인트를 공유하고 안전하게 병합한다.
- 당장 팀에서 바로 사용할 수 있는 최소 규칙부터 먼저 정한다.

---

## 왜 브랜치 / Issue / PR을 나누는가

일반적으로 협업 개발에서는 아래 이유 때문에 작업을 분리한다.

1. `Issue`: 왜 이 작업을 하는지, 무엇을 해야 하는지 기록한다.
2. `Branch`: 메인 브랜치에 직접 영향을 주지 않고 독립적으로 작업한다.
3. `PR`: 변경 내용을 검토하고 테스트 결과를 공유한 뒤 병합한다.

이 흐름을 따르면 다음이 쉬워진다.

- 작업 단위 추적
- 리뷰 포인트 공유
- 충돌 최소화
- 회고 및 이력 관리

---

## 저장소 단위 원칙

현재 작업 디렉터리 `MMP`는 하나의 Git 저장소가 아니다.

- `client`: 프론트엔드 구현 작업
- `mmp-server`: 백엔드 API 및 서버 구현 작업
- `docs`: 스터디 문서, 회의록, 협업 문서

따라서 아래 원칙을 따른다.

1. 저장소별로 브랜치와 PR을 분리한다.
2. 한 PR에 서로 다른 저장소의 변경을 섞지 않는다.
3. FE 구현, BE 구현, 문서 작업은 가능하면 별도 Issue와 PR로 나눈다.

예시

- 프론트 화면 수정: `client`에서 브랜치 생성 및 PR
- 백엔드 API / 서버 수정: `mmp-server`에서 브랜치 생성 및 PR
- 문서 정리: `docs`에서 브랜치 생성 및 PR

---

## 작업 단위 원칙

한 브랜치와 한 PR은 가능한 한 하나의 목적만 다룬다.

- 좋은 예
  - 대시보드 카메라 월 레이아웃 개선
  - 스트림 상태 조회 API 추가
  - 장치 목록 응답 타입 정리
  - PR / Issue 컨벤션 문서 추가
- 피해야 하는 예
  - 대시보드 개편 + 문서 작성 + 패키지 업데이트 + 로그 API 수정

작업 단위를 나눌 때는 아래 기준을 우선한다.

1. 사용자 입장에서 하나의 기능인지
2. 리뷰어가 한 번에 이해할 수 있는 범위인지
3. 실패했을 때 롤백하기 쉬운지

---

## 브랜치 네이밍 규칙

형식:

```text
<type>/#<issue-number>-<short-description>
```

사용할 `type`

- `feat`: 새로운 기능 추가
- `fix`: 버그 수정
- `refactor`: 동작 변화 없는 구조 개선
- `docs`: 문서 작업
- `chore`: 설정, 빌드, 패키지, 관리성 작업

예시

```text
feat/#12-dashboard-camera-wall
feat/#24-stream-status-api
fix/#31-device-list-null-guard
refactor/#33-monitoring-layout-store
docs/#21-branch-issue-pr-convention
chore/#40-eslint-config-cleanup
```

짧은 설명 규칙

- 소문자 `kebab-case` 사용
- 너무 길게 쓰지 않는다
- 작업 목적이 바로 드러나게 작성한다

---

## Issue 작성 규칙

### 제목 형식

```text
[FE] <작업 주제>
[BE] <작업 주제>
[DOCS] <작업 주제>
```

예시

- `[FE] 모니터링 대시보드 카메라 월 레이아웃 개선`
- `[FE] History / System Logs 무스크롤 레이아웃 정리`
- `[BE] 스트림 상태 조회 API 추가`
- `[BE] 카메라 메타데이터 응답 구조 정리`
- `[DOCS] 브랜치 / Issue / PR 컨벤션 초안 정리`

### Issue 본문 템플릿

```md
## 배경
- 왜 이 작업이 필요한지 작성

## 목표
- 이번 작업에서 달성해야 할 핵심 목표 작성

## 작업 범위
- 포함되는 작업 1
- 포함되는 작업 2
- 포함되지 않는 작업 1

## 세부 작업
- [ ] 세부 작업 1
- [ ] 세부 작업 2
- [ ] 세부 작업 3

## 완료 조건
- 완료로 판단할 기준 1
- 완료로 판단할 기준 2

## 참고
- 피그마 링크
- 회의록 링크
- 관련 API 문서
- 화면 캡처 또는 예시 데이터
```

Issue에는 최소한 아래 네 가지가 있어야 한다.

1. 왜 하는지
2. 어디까지 하는지
3. 무엇을 체크해야 하는지
4. 언제 완료로 보는지

---

## PR 제목 규칙

형식:

```text
[FE] #<issue-number> <작업 주제>
[BE] #<issue-number> <작업 주제>
[DOCS] #<issue-number> <작업 주제>
```

예시

- `[FE] #12 대시보드 카메라 월 레이아웃 개선`
- `[FE] #13 서브 페이지 무스크롤 레이아웃 정리`
- `[BE] #24 스트림 상태 조회 API 추가`
- `[BE] #25 녹화 세그먼트 정리 작업 스케줄러 구현`
- `[DOCS] #21 브랜치 / Issue / PR 컨벤션 초안 추가`

---

## PR 본문 템플릿

```md
## 작업 개요
- 이번 PR에서 해결한 내용을 2~3줄로 정리

## 관련 Issue
- Closes #이슈번호

## 변경 사항
- 변경 사항 1
- 변경 사항 2
- 변경 사항 3

## 테스트
- [ ] 빌드 확인
- [ ] 로컬 동작 확인
- [ ] 필요한 경우 API 응답 확인
- [ ] 필요한 경우 화면 확인

## 체크리스트
- [ ] 한 PR에 하나의 목적만 담았다
- [ ] 불필요한 파일 변경이 없다
- [ ] 리뷰어가 확인해야 할 포인트를 적었다
- [ ] 관련 문서나 타입 변경이 필요한지 확인했다

## 리뷰 포인트
- 집중해서 봐야 할 부분 1
- 집중해서 봐야 할 부분 2

## 참고 자료
- 스크린샷, GIF, API 예시 응답, 로그 등
```

PR에는 구현 결과만 적는 것이 아니라, 리뷰어가 무엇을 봐야 하는지도 적는다.

---

## 커밋 메시지 규칙

형식:

```text
<type>: <summary>
```

예시

```text
feat: add camera wall layout for monitoring view
feat: compress playback layout without internal scroll
feat: add stream status endpoint
fix: guard empty device response
docs: add branch issue pr workflow draft
```

권장 사항

- 한 커밋에 하나의 의미만 담는다
- 기능 단위 또는 화면 단위로 자른다
- `wip`, `test`, `fix2` 같은 메시지는 피한다

---

## FE / BE 작업 분리 예시

### 1. 프론트엔드 화면 작업

- Issue 제목
  - `[FE] 모니터링 대시보드 카메라 월 레이아웃 개선`
- 브랜치명
  - `feat/#12-dashboard-camera-wall`
- 범위
  - 카메라 월 UI 수정
  - 카드 배치 조정
  - 내부 스크롤 제거

### 2. 백엔드 API 작업

- Issue 제목
  - `[BE] 스트림 상태 조회 API 추가`
- 브랜치명
  - `feat/#24-stream-status-api`
- 범위
  - 스트림 상태 응답 스키마 정의
  - API 엔드포인트 구현
  - 예외 응답 처리

### 3. 문서 작업

- Issue 제목
  - `[DOCS] 브랜치 / Issue / PR 컨벤션 초안 정리`
- 브랜치명
  - `docs/#21-branch-issue-pr-convention`
- 범위
  - 협업 규칙 정리
  - 템플릿 초안 작성

---

## 권장 작업 흐름

1. Issue 생성
2. 작업 저장소 최신화
3. Issue 번호를 포함한 브랜치 생성
4. 작업 후 의미 단위로 커밋
5. 빌드, 테스트, 로컬 확인
6. 원격 브랜치 push
7. Issue 번호를 연결한 PR 생성
8. 리뷰 반영 후 merge

### `client` 작업 예시

```bash
cd client
git checkout main
git pull origin main
git checkout -b feat/#12-dashboard-camera-wall
```

### `docs` 작업 예시

```bash
cd docs
git checkout main
git pull origin main
git checkout -b docs/#21-branch-issue-pr-convention
```

### `mmp-server` 작업 예시

```bash
cd mmp-server
git checkout main
git pull origin main
git checkout -b feat/#24-stream-status-api
```

---

## 추후 정리하면 좋은 항목

- GitHub Labels 규칙
- Issue Template 실제 파일화
- `.github/PULL_REQUEST_TEMPLATE.md` 추가
- 리뷰어 지정 규칙
- squash merge 여부
- PR 크기 제한 기준

---

## 현재 단계에서의 제안

지금은 아래 4가지만 먼저 팀 규칙으로 합의하면 충분하다.

1. 작업은 Issue로 먼저 정의한다
2. 구현은 브랜치를 분리해서 진행한다
3. PR에는 `관련 Issue`, `변경 사항`, `테스트`, `리뷰 포인트`를 반드시 적는다
4. 서로 다른 목적의 작업은 한 PR에 섞지 않는다

이 문서는 초안이며, 실제 사용하면서 불편한 부분이 생기면 팀 규칙에 맞게 축약하거나 보완한다.
