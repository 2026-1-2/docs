# 프롬프트
> MediaMTX에서 HLS 폴백을 설정하는 방법을 설명해줘.

## HLS 설정 옵션

```yaml
hlsAlwaysRemux: yes      # 항상 HLS 세그먼트 생성
hlsSegmentCount: 3       # 유지할 세그먼트 수
hlsSegmentDuration: 2s   # 세그먼트 1개 길이
```

---

## 동작 방식

```
hlsAlwaysRemux: yes 설정 시
RTSP 수신 즉시
  → WebRTC SRTP 패킷 생성
  → HLS 세그먼트 동시 생성 (병렬)
```

---

## 세그먼트 버퍼 계산

```
세그먼트 수(3) × 길이(2초) = 6초 버퍼
```

---

## 트레이드오프

| 설정 | 효과 |
|---|---|
| segmentDuration 줄이기 | 지연 감소, 서버 부하 증가 |
| segmentCount 늘리기 | 안정성 증가, 메모리 사용 증가 |
