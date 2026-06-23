# recording_segments 테이블을 별도로 분리한 이유

## recording 테이블 하나로 했을 때의 문제

```sql
-- 단순 구조: recording 하나로 전부
CREATE TABLE recordings (
  id INT PRIMARY KEY,
  camera_id INT,
  started_at DATETIME,
  ended_at DATETIME,
  file_path VARCHAR(500)  -- 하나의 파일 경로만 저장
);
```

HLS 녹화는 **하나의 녹화 세션이 수백 개의 .ts 세그먼트 파일로 쪼개진다.**

```
recording 1 (2시간 녹화)
  ├── segment_0000.ts  (2초)
  ├── segment_0001.ts  (2초)
  ├── segment_0002.ts  (2초)
  ...
  └── segment_3599.ts  (2초)
```

`file_path` 하나로는 이 구조를 담을 수 없다.

---

## 1:N 분리의 실제 이유

### 이유 1: 파일 단위 삭제

보관 정책(storage_policies)은 "30일 이전 파일 삭제"처럼 세그먼트 단위로 동작한다.
세그먼트를 개별 행으로 관리해야 특정 세그먼트만 골라서 지울 수 있다.

```sql
-- 30일 이전 세그먼트만 삭제 (전체 녹화가 아닌 세그먼트 단위)
DELETE FROM recording_segments
WHERE created_at < NOW() - INTERVAL 30 DAY;
```

recording 테이블 하나로 했다면 세그먼트 단위 삭제가 불가능하다.

### 이유 2: 세그먼트별 상태 추적

```sql
CREATE TABLE recording_segments (
  id          INT PRIMARY KEY AUTO_INCREMENT,
  recording_id INT NOT NULL,
  file_path   VARCHAR(500) NOT NULL,
  sequence_no INT NOT NULL,       -- 순서
  duration_sec FLOAT,             -- 재생 시간
  file_size_bytes BIGINT,         -- 용량
  created_at  DATETIME NOT NULL,
  FOREIGN KEY (recording_id) REFERENCES recordings(id) ON DELETE CASCADE
);
```

세그먼트마다 용량, 재생 시간, 순서를 개별 관리한다.
전체 녹화 용량 합산, 특정 구간 파일 탐색이 가능해진다.

### 이유 3: 부분 재생

특정 시각의 영상을 찾을 때 세그먼트 테이블을 조회하면 정확한 파일을 특정할 수 있다.

```sql
-- 특정 시각(2026-06-21 15:30:00)의 세그먼트 찾기
SELECT s.file_path
FROM recording_segments s
JOIN recordings r ON s.recording_id = r.id
WHERE r.camera_id = 1
  AND s.created_at <= '2026-06-21 15:30:00'
ORDER BY s.created_at DESC
LIMIT 1;
```

recording 하나에 뭉쳐있었다면 파일 오프셋 계산을 직접 해야 한다.

---

## 테이블 관계

```
recordings (1)
  id, camera_id, started_at, ended_at, status

      ↓ 1:N

recording_segments (N)
  id, recording_id, file_path, sequence_no, duration_sec, file_size_bytes, created_at
```

---

## 한 줄 요약

HLS 녹화의 물리적 단위(세그먼트 파일)와 논리적 단위(녹화 세션)가 다르기 때문에 테이블을 분리한다.
`recording`은 "언제부터 언제까지 녹화했는가", `recording_segments`는 "그 녹화를 구성하는 파일들"이다.
