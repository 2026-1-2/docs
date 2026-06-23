# FFmpeg → OpenCV로 대체 가능하다는 말의 의미

## 맥락: 스냅샷(썸네일) 생성

스트리밍 중인 카메라에서 현재 프레임을 JPG 이미지로 뽑아내는 기능이다.
아키텍처 노트에 "FFmpeg 대신 OpenCV로 대체 가능"이라고 적혀 있는 부분이다.

---

## 두 방법 모두 같은 출력을 만든다

### FFmpeg로 스냅샷 뽑기
```bash
ffmpeg -rtsp_transport tcp -i rtsp://camera/stream -frames:v 1 /snapshots/cam1.jpg
```

### OpenCV로 스냅샷 뽑기 (Python 예시)
```python
import cv2
cap = cv2.VideoCapture("rtsp://camera/stream")
ret, frame = cap.read()
cv2.imwrite("/snapshots/cam1.jpg", frame)
cap.release()
```

**결과는 동일하다: `/snapshots/cam1.jpg`**

---

## 핵심: 출력 파일 경로가 같으면 뒷단 로직이 동일하다

```
FFmpeg or OpenCV
      ↓
  cam1.jpg 생성
      ↓
  chokidar 감지 (파일 변경 감지)
      ↓
  NestJS에게 이벤트 전달
      ↓
  DB 업데이트 or SSE로 React에 알림
```

chokidar는 파일시스템을 감시한다. `/snapshots/` 디렉토리에 `.jpg`가 생기면 감지한다.
그 파일을 **누가 만들었는지는 chokidar가 신경 쓰지 않는다.**

---

## 그러면 왜 대체 가능하다고 하는가?

| | FFmpeg | OpenCV |
|--|--------|--------|
| 의존성 | 바이너리 설치 필요 | Python/C++ 라이브러리 |
| 활용 범위 | 단순 프레임 추출 | 프레임 추출 + AI 분석 |
| 커스터마이징 | 제한적 | 자유로움 |
| 속도 | 빠름 | 비슷하거나 느림 |

나중에 스냅샷에 **객체 감지나 모션 분석**을 붙이고 싶으면 OpenCV로 교체하면 된다.
chokidar와 NestJS 이후 로직은 건드릴 필요가 없다.

---

## 결론

"대체 가능하다"는 말은 인터페이스(출력 파일)가 같기 때문에 구현체를 바꿀 수 있다는 뜻이다.
파일을 만들어내는 도구가 무엇이든, 뒷단의 chokidar → NestJS → React 파이프라인은 동일하게 동작한다.
