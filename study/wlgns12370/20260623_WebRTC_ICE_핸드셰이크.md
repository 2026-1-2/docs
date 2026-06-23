# 프롬프트
> WebRTC ICE 핸드셰이크 과정을 단계별로 설명해줘.

## ICE란

WebRTC에서 두 피어 간 최적의 네트워크 경로를 찾는 프로토콜.

---

## 연결 수립 3단계

### Step 1 — Offer/Answer (SDP 교환)
```
React Client → POST :8889 → SDP Offer 전송
MediaMTX → SDP Answer 반환
```

### Step 2 — ICE Candidate 교환
- 양측이 가능한 네트워크 경로를 서로 공유
- 최적 경로 자동 선택

### Step 3 — 미디어 전송
```
DTLS 암호화 SRTP 스트림 → UDP:8189
방화벽 환경 → TCP:8189 자동 폴백
```

---

## 포트 역할

| 포트 | 용도 |
|---|---|
| :8889 | SDP Offer/Answer 시그널링 |
| :8189/udp | ICE 미디어 전송 주경로 |
| :8189/tcp | ICE TCP Fallback |
