# 프롬프트
> WebRTC에서 STUN과 TURN 서버의 역할 차이를 설명해줘. MMP가 외부 서버 없이 동작하는 이유도 포함해서.

## STUN 서버

**Session Traversal Utilities for NAT** — 자신의 공인 IP/포트를 알아내는 서버.

```
클라이언트 → STUN 서버 → "당신의 공인 IP는 1.2.3.4, 포트는 54321입니다"
```

---

## TURN 서버

**Traversal Using Relays around NAT** — 직접 연결이 불가할 때 트래픽을 중계하는 서버.

```
클라이언트 A → TURN 서버 → 클라이언트 B
```

방화벽이 매우 엄격한 환경에서만 필요. 트래픽 전체가 TURN을 거치므로 서버 부하 높음.

---

## MMP가 외부 STUN/TURN 없이 동작하는 이유

MMP는 **내부망 온프레미스** 환경에서 운영된다.

```
관리자 PC (192.168.0.50)
    ↕ 내부망 (NAT 없음)
MMP 서버 (192.168.0.100)
```

- 동일 내부망 → NAT 통과 불필요 → STUN 불필요
- 직접 연결 가능 → TURN 불필요
- `PUBLIC_HOST`에 내부 IP 지정으로 ICE Candidate 생성

---

## 외부 접근이 필요한 경우

```env
PUBLIC_HOST=mmp.example.com  # 외부 도메인 지정
# + STUN 서버 설정 추가 필요
```
