# 프롬프트
> MMP의 이중 토큰(Dual-Token) JWT 전략을 설명해줘.

## 단일 토큰의 문제

토큰 탈취 시 만료 전까지 공격자가 계속 사용 가능.

---

## Dual-Token 전략

```
Access Token  : 유효기간 짧음 (예: 15분)
Refresh Token : 유효기간 김 (예: 7일)
```

### 동작 흐름

```
1. 로그인 → Access Token + Refresh Token 발급
2. API 요청 → Authorization: Bearer {Access Token}
3. Access Token 만료 → Refresh Token으로 갱신
4. Refresh Token 만료 → 재로그인 요구
```

---

## MMP 구현

```env
JWT_SECRET=...          # Access Token 서명
JWT_REFRESH_SECRET=...  # Refresh Token 서명 (별도)
```

두 토큰을 다른 시크릿으로 분리 서명 → 탈취 시 피해 범위 최소화.
