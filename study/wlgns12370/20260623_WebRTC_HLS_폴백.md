# 프롬프트
> WebRTC 연결 실패 시 HLS 폴백이 어떻게 동작하는지 설명해줘.

## 왜 폴백이 필요한가

일부 방화벽은 UDP를 차단 → WebRTC 연결 불가.  
폴백 없으면 해당 환경에서 스트리밍 자체가 불가능.

---

## HLS 폴백 설정

```yaml
# mediamtx.yml
hlsAlwaysRemux: yes      # WebRTC 여부와 무관하게 HLS 세그먼트 항상 생성
hlsSegmentCount: 3
hlsSegmentDuration: 2s   # → 약 6초 버퍼
```

---

## 폴백 흐름

```
1. WebRTC 연결 시도 (:8889 시그널링 → :8189 미디어)
2. 성공 → 0.1~0.5초 지연 스트리밍
3. 실패 → HLS 폴백으로 자동 전환 (~6초 지연)
```
