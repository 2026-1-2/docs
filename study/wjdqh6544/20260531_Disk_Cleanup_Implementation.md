## ChatGPT 5.5 (2026. 05. 31.)
## Question
```
용량이 없는 경우 저장된 영상 중 오래된 것부터 제거하는 기능은 어떻게 구현을 하지?

현재 Recorder 는 파이썬으로 구현되어 있음.
```
## Answer
이건 Recorder 내부에서 관리하는 것이 가장 일반적입니다.

보통 NVR, DVR, CCTV 녹화 시스템들은 **"보존 기간 기반"** 또는 **"디스크 사용량 기반"** 삭제 정책을 사용합니다.

### 가장 쉬운 방법: 디스크 사용량 기반

예를 들어:

```text
/storage
├── camera1
│   ├── 2026-06-01-10.mp4
│   ├── 2026-06-01-11.mp4
│   └── ...
├── camera2
└── ...
```

Recorder가 주기적으로:

```python
import shutil

usage = shutil.disk_usage("/storage")

used_percent = usage.used / usage.total * 100

if used_percent > 90:
    delete_oldest_files()
```

를 수행합니다.

---

### 오래된 파일 찾기

전체 녹화 파일을 수집해서 생성시간 기준 정렬:

```python
from pathlib import Path

files = []

for f in Path("/storage").rglob("*.mp4"):
    files.append(f)

files.sort(key=lambda x: x.stat().st_mtime)
```

가장 오래된 파일:

```python
oldest = files[0]
oldest.unlink()
```

---

### 실제 운영 방식

보통은:

```text
90% 도달
↓
삭제 시작
↓
85% 이하까지 삭제
↓
중지
```

처럼 히스테리시스를 둡니다.

예:

```python
HIGH_WATERMARK = 90
LOW_WATERMARK = 85
```

이유는 삭제 직후 다시 임계치에 걸려 반복 실행되는 것을 막기 위해서입니다.

---

### 카메라별 공평성 문제

단순히 오래된 파일부터 삭제하면:

```text
camera1
  100GB

camera2
  5GB
```

인 경우 camera1 영상만 계속 지워질 수 있습니다.

CCTV 계열에서는 오히려 이게 정상인 경우가 많습니다.

정책은 보통:

> 가장 오래된 영상부터 삭제

입니다.

즉 카메라 구분 없이 시간순으로 삭제합니다.

---

### 파일명에 시간이 포함되어 있다면

예:

```text
camera1/
  20260601_120000.mp4
  20260601_121000.mp4
```

라면 stat() 대신 파일명만으로도 정렬 가능합니다.

```python
sorted(files)
```

하지만 실제로는 mtime 기준이 더 안전합니다.

---

### 더 좋은 방법: 녹화 전에 체크

녹화 프로세스가 새 파일을 만들기 직전에:

```python
ensure_free_space()
create_new_segment()
```

처럼 구현합니다.

예:

```python
def ensure_free_space():
    while disk_usage_percent() > 90:
        delete_oldest_file()
```

그러면 디스크가 꽉 차서 녹화 실패하는 상황을 예방할 수 있습니다.

---

### 추천 구조

Recorder가 Python이라면 별도 서비스 없이 내부에 다음 스레드 하나 추가하는 게 가장 간단합니다.

```python
def storage_manager():
    while True:
        if disk_usage_percent() > 90:
            purge_old_files_until(85)

        time.sleep(60)
```

Recorder 프로세스:

```text
Recorder
├── Camera Worker 1
├── Camera Worker 2
├── Camera Worker 3
└── Storage Manager
```

이 구조가 구현 난이도도 낮고 운영 경험상 가장 안정적입니다.

추가로 녹화 규모가 크다면(수십~수백 채널), 파일을 매번 `rglob()`으로 전수조사하지 말고 SQLite에 녹화 파일 목록(`path`, `size`, `created_at`)을 기록해 두고 가장 오래된 항목을 조회해서 삭제하는 방식이 훨씬 효율적입니다. 카메라 수가 몇 개 정도인지에 따라 권장 방식이 달라집니다.


---------------
## Question
```
disk_usage_percent 및 purge_old_files_until 함수를 작성해주기 바람.
```
## Answer
아래 정도면 실무에서 바로 사용할 수 있는 수준입니다.

```python
from pathlib import Path
import shutil
import logging

logger = logging.getLogger(__name__)


def disk_usage_percent(storage_path: str) -> float:
    """
    지정된 경로가 위치한 파일시스템의 사용률(%)
    """
    usage = shutil.disk_usage(storage_path)

    return (usage.used / usage.total) * 100.0


def purge_old_files_until(
    storage_path: str,
    target_percent: float,
    extensions: tuple[str, ...] = (".mp4", ".mkv", ".ts"),
) -> int:
    """
    디스크 사용량이 target_percent 이하가 될 때까지
    가장 오래된 파일부터 삭제한다.

    Returns:
        삭제된 파일 수
    """

    files = []

    for ext in extensions:
        files.extend(Path(storage_path).rglob(f"*{ext}"))

    files = [f for f in files if f.is_file()]

    files.sort(key=lambda f: f.stat().st_mtime)

    deleted_count = 0

    for file_path in files:

        current_usage = disk_usage_percent(storage_path)

        if current_usage <= target_percent:
            break

        try:
            size_mb = file_path.stat().st_size / (1024 * 1024)

            logger.warning(
                "Deleting old recording: %s (%.1f MB)",
                file_path,
                size_mb,
            )

            file_path.unlink()

            deleted_count += 1

        except Exception:
            logger.exception(
                "Failed to delete recording: %s",
                file_path,
            )

    logger.info(
        "Purge completed. deleted=%d usage=%.2f%%",
        deleted_count,
        disk_usage_percent(storage_path),
    )

    return deleted_count
```

사용 예시:

```python
HIGH_WATERMARK = 90
LOW_WATERMARK = 85

usage = disk_usage_percent("/recordings")

if usage > HIGH_WATERMARK:
    purge_old_files_until(
        "/recordings",
        LOW_WATERMARK,
    )
```

백그라운드 스레드:

```python
import time
import threading


def storage_manager():
    while True:
        try:
            usage = disk_usage_percent("/recordings")

            if usage > 90:
                purge_old_files_until(
                    "/recordings",
                    85,
                )

        except Exception:
            logger.exception("storage manager error")

        time.sleep(60)


threading.Thread(
    target=storage_manager,
    daemon=True,
).start()
```

추가로 녹화 중인 파일을 삭제하지 않도록 보호하는 것이 중요합니다.

예를 들어 ffmpeg가:

```text
camera1/
 ├── current.mp4      <- 현재 기록 중
 ├── 20260601_1200.mp4
 └── 20260601_1210.mp4
```

형태라면 `current.mp4`는 삭제 대상에서 제외해야 합니다.

보통은 세그먼트 녹화를 사용해서:

```text
camera1/
 ├── 20260601_1200.mp4
 ├── 20260601_1210.mp4
 ├── 20260601_1220.mp4
```

처럼 완성된 파일만 생성되게 하면 가장 안전합니다. 이 경우 위 코드 그대로 사용해도 문제가 거의 없습니다.

--------------------
## Question
```
특정 카메라의 영상을 지우면, 다른 카메라의 오래된 영상도 삭제되도록 하고 싶음.
```
## Answer
그렇다면 **"가장 오래된 파일부터 전역(Global) 기준으로 삭제"** 정책을 쓰면 됩니다.

예를 들어:

```text
/storage
├── camera1
│   ├── 10:00.mp4
│   ├── 10:10.mp4
│   └── 10:20.mp4
├── camera2
│   ├── 10:05.mp4
│   ├── 10:15.mp4
│   └── 10:25.mp4
```

정렬하면:

```text
camera1/10:00.mp4
camera2/10:05.mp4
camera1/10:10.mp4
camera2/10:15.mp4
...
```

삭제 순서도 동일합니다.

즉:

* camera1 삭제
* 다음은 camera2 삭제
* 다시 camera1 삭제

처럼 자연스럽게 섞입니다.

사실 제가 앞서 제안한 `purge_old_files_until()`도 이미 이 방식입니다.

```python
files.sort(
    key=lambda f: f.stat().st_mtime
)
```

여기서 카메라 디렉토리를 구분하지 않고 전체 파일을 한 리스트에 넣은 뒤 정렬하므로:

```python
[
    camera1/file1,
    camera2/file1,
    camera1/file2,
    camera3/file1,
]
```

형태가 됩니다.

---

만약 파일 생성 시각이 아닌 **녹화 시작 시각**으로 정확하게 관리하고 싶다면 파일명에 타임스탬프를 넣는 것이 더 좋습니다.

예:

```text
camera1/20260601_100000.mp4
camera2/20260601_100500.mp4
camera1/20260601_101000.mp4
```

그러면:

```python
files.sort(key=lambda p: p.name)
```

만으로도 시간순 정렬이 가능합니다.

---

다만 질문을 읽어보니 의도가 조금 다를 수도 있습니다.

### 정책 A (권장)

> 저장소 전체를 하나로 보고 가장 오래된 영상부터 삭제

```text
camera1 100GB
camera2 100GB
camera3 100GB
```

→ 전 카메라를 대상으로 시간순 삭제

대부분의 CCTV/NVR 제품이 이 방식입니다.

---

### 정책 B

> camera1의 새 영상이 들어오려고 할 때 camera1 영상만 삭제

```text
camera1 꽉 참
↓
camera1 오래된 영상 삭제
```

이 경우 카메라별 할당량(quota)을 관리해야 합니다.

---

### 정책 C

> camera1이 새 영상을 쓰려다가 공간이 부족하면
> camera2, camera3의 오래된 영상도 삭제 가능

이건 결국 정책 A와 같습니다.

저장소를 공유하는 이상:

```text
전체 사용률 > 90%
↓
전체 파일 중 가장 오래된 것 삭제
```

가 가장 합리적이고 구현도 단순합니다.

따라서 원하시는 것이 "어느 카메라 때문에 용량이 부족해졌든, 모든 카메라를 대상으로 가장 오래된 영상부터 지운다"라면 현재의 전역 정렬 + 전역 삭제 방식이 정확히 그 동작입니다.

--------------
## Question
```
video_path 는 전역 directory 가 주어짐.
그러니까, 
video_path/cam1/20260601_111552.ts
video_path/cam2/20260601_111552.ts
video_path/cam3/20260601_111552.ts

이런 식으로 영상이 저장됨. f.stat().st_mtime 을 쓰는 경우, cam1/2/3 에 관계없이 내가 원하는 대로 구현이 되나?
```
## Answer
네. 현재 구조라면 `f.stat().st_mtime`으로 정렬하면 카메라 디렉토리와 무관하게 전체 파일을 시간순으로 정렬할 수 있습니다.

예를 들어:

```text
/video
├── cam1
│   └── 20260601_111552.ts   (11:15:52)
├── cam2
│   └── 20260601_111553.ts   (11:15:53)
├── cam3
│   └── 20260601_111554.ts   (11:15:54)
```

코드:

```python
files = list(Path(video_path).rglob("*.ts"))

files.sort(key=lambda f: f.stat().st_mtime)
```

결과:

```python
[
    Path("/video/cam1/20260601_111552.ts"),
    Path("/video/cam2/20260601_111553.ts"),
    Path("/video/cam3/20260601_111554.ts"),
]
```

즉 삭제 순서는:

```text
cam1 파일 삭제
cam2 파일 삭제
cam3 파일 삭제
```

가 됩니다.

---

다만 CCTV 녹화 시스템에서는 `st_mtime`보다 **파일명 기반 정렬**을 더 추천합니다.

왜냐하면 `st_mtime`은 다음과 같은 상황에서 변경될 수 있습니다.

```text
- rsync 복사
- 백업/복원
- chmod/chown
- 컨테이너 마이그레이션
- 파일 이동
```

반면 현재 파일명이:

```text
20260601_111552.ts
20260601_111553.ts
```

처럼 녹화 시작 시간을 포함하고 있으므로, 파일명 자체가 더 신뢰할 수 있는 "녹화 시각"입니다.

예:

```python
from pathlib import Path

files = [
    f
    for f in Path(video_path).rglob("*.ts")
    if f.is_file()
]

files.sort(key=lambda f: f.name)
```

또는 명시적으로:

```python
files.sort(
    key=lambda f: f.stem  # 20260601_111552
)
```

---

더 좋은 방법은 카메라 이름과 함께 정렬하는 것이 아니라 **타임스탬프만 추출해서 정렬**하는 것입니다.

```python
from datetime import datetime

def recording_time(path: Path):
    return datetime.strptime(
        path.stem,
        "%Y%m%d_%H%M%S",
    )

files.sort(key=recording_time)
```

그러면:

```text
cam1/20260601_111552.ts
cam2/20260601_111553.ts
cam3/20260601_111554.ts
```

뿐 아니라

```text
cam1/20260601_111552.ts
cam2/20260601_111552.ts
cam3/20260601_111552.ts
```

처럼 동일 시각 파일도 모두 같은 시점으로 취급됩니다.

---

정리하면:

* `rglob()` + `st_mtime` → 카메라 디렉토리 무시하고 전역 시간순 삭제 가능
* 하지만 파일명에 `YYYYMMDD_HHMMSS.ts`가 들어있다면 **파일명 기준 정렬이 더 안정적**
* CCTV/NVR 계열에서는 보통 파일명(녹화 시작 시각) 기준으로 삭제 정책을 구현합니다.


-------------
## Question
```
def purge_old_files_until(video_path: str, extensions: list[str]=[".ts"], target_percent: float):
    files = []
    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))
    
    files = [f for f in files if f.is_file()]
    
    files.sort(key=recording_time)
    
    del_count = 0
    
    for file_path in files:
        cur_usage = disk_usage_percent(video_path=video_path)
        if (cur_usage <= target_percent):
            break
            
        try:
            size_mb = file_path.stat().st_size / (1024 * 1024)
            logger.warning("Deleting old recording: %s (%.1f MB)",)

그러면 이걸 어떻게 고쳐야 하지?
삭제가 한 번 발생할 때, 카메라 대수만큼 삭제하도록 Operation 이 진행되어야 할 것 같은데
```
## Answer
지금 요구사항을 보면 단순히

> 가장 오래된 파일 1개 삭제

가 아니라

> 삭제가 시작되면 각 카메라의 오래된 파일도 함께 삭제

를 원하시는 것 같습니다.

예를 들어:

```text
cam1/20260601_111552.ts
cam2/20260601_111553.ts
cam3/20260601_111554.ts
cam1/20260601_112552.ts
cam2/20260601_112553.ts
cam3/20260601_112554.ts
```

현재 코드라면:

```text
삭제 #1
→ cam1/20260601_111552.ts
```

만 삭제됩니다.

원하시는 것은 아마:

```text
삭제 Cycle #1
→ cam1/20260601_111552.ts
→ cam2/20260601_111553.ts
→ cam3/20260601_111554.ts
```

즉 "가장 오래된 세그먼트 세트"를 한 번에 제거하는 것입니다.

---

## 방법 1: 카메라별 가장 오래된 파일 삭제 (추천)

카메라별로 파일을 그룹화합니다.

```python
from collections import defaultdict

camera_files = defaultdict(list)

for f in files:
    camera = f.parent.name
    camera_files[camera].append(f)

for camera in camera_files:
    camera_files[camera].sort(key=recording_time)
```

삭제 Cycle:

```python
while disk_usage_percent(video_path) > target_percent:

    deleted = False

    for camera, file_list in camera_files.items():

        if not file_list:
            continue

        oldest = file_list.pop(0)

        try:
            oldest.unlink()
            deleted = True

        except Exception:
            logger.exception(
                "failed deleting %s",
                oldest,
            )

    if not deleted:
        break
```

결과:

```text
1차
cam1 oldest 삭제
cam2 oldest 삭제
cam3 oldest 삭제

2차
cam1 second 삭제
cam2 second 삭제
cam3 second 삭제
```

매우 공평합니다.

---

## 방법 2: 동일 timestamp 파일 묶음 삭제

만약 파일명이

```text
cam1/20260601_111552.ts
cam2/20260601_111552.ts
cam3/20260601_111552.ts
```

처럼 동일한 시간 단위로 생성된다면 더 깔끔합니다.

그룹화:

```python
from collections import defaultdict

groups = defaultdict(list)

for f in files:
    groups[f.stem].append(f)
```

예:

```python
{
  "20260601_111552": [
      cam1/...
      cam2/...
      cam3/...
  ],
  "20260601_112552": [...]
}
```

정렬:

```python
timestamps = sorted(groups.keys())
```

삭제:

```python
for ts in timestamps:

    if disk_usage_percent(video_path) <= target_percent:
        break

    for file_path in groups[ts]:
        file_path.unlink()
```

결과:

```text
20260601_111552 그룹 삭제
 ├─ cam1
 ├─ cam2
 └─ cam3

20260601_112552 그룹 삭제
 ├─ cam1
 ├─ cam2
 └─ cam3
```

---

현재 구조가

```text
video_path/
  cam1/
    20260601_111552.ts
  cam2/
    20260601_111552.ts
  cam3/
    20260601_111552.ts
```

처럼 **모든 카메라가 같은 시각으로 세그먼트를 생성한다면**, 저는 **방법 2(타임스탬프 그룹 삭제)** 를 추천합니다.

이게 의미적으로도:

> "2026-06-01 11:15:52 시점의 녹화 데이터"

를 통째로 제거하는 것이어서 CCTV 시스템 관점에서 가장 자연스럽습니다.

-----------------
## Question
```
이렇게 하면 되나?

def disk_usage_percent(video_path: str):
    usage = shutil.disk_usage(video_path)
    return (usage.used / usage.total) * 100.0

def recording_time(path: Path):
    return datetime.strptime(path.stem, "%Y%m%d_%H%M%S")

def purge_old_files_until(video_path: str, target_percent: float, extensions: list[str]=[".ts"]):
    camera_files = defaultdict(list)
    files = []
    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))
    
    files = [f for f in files if f.is_file()]
    
    for f in files:
        camera = f.parent.name
        camera_files[camera].append(f)
    
    for camera in camera_files:
        camera_files.sort(key=recording_time)
    
    while disk_usage_percent(video_path=video_path) > target_percent:
        deleted = False
        
        for _, file_list in camera_files.items():
            if not file_list:
                continue
            
            oldest = file_list.pop()
            try:
                oldest.unlink()
                deleted=True
            except Exception:
                logger.exception("Failed Deleting %s", oldest)
                
            if not deleted:
                break
```
## Answer
거의 맞는데 몇 군데 버그가 있습니다.

### 1. `camera_files.sort()` 버그

현재:

```python
for camera in camera_files:
    camera_files.sort(key=recording_time)
```

`camera_files`는 `dict`라서 `sort()`가 없습니다.

아래처럼 해야 합니다.

```python
for camera in camera_files:
    camera_files[camera].sort(key=recording_time)
```

---

### 2. `pop()` 방향이 반대

현재:

```python
oldest = file_list.pop()
```

정렬이 오름차순이면:

```text
[
  11:00,
  11:10,
  11:20
]
```

`pop()`은 마지막 요소를 꺼내므로

```text
11:20
```

가 삭제됩니다.

즉 가장 최신 파일을 지우게 됩니다.

원하는 것은:

```python
oldest = file_list.pop(0)
```

입니다.

---

### 3. `deleted` 체크 위치

현재:

```python
for _, file_list in camera_files.items():
    ...
    if not deleted:
        break
```

이건 의도와 다르게 동작합니다.

삭제 실패한 카메라가 하나 있어도 다른 카메라는 계속 시도하는 게 보통 더 낫습니다.

오히려 바깥 루프 종료용으로 사용해야 합니다.

```python
while disk_usage_percent(video_path) > target_percent:
    deleted = False

    for _, file_list in camera_files.items():
        ...
        oldest.unlink()
        deleted = True

    if not deleted:
        break
```

---

### 4. 빈 디렉토리 정리 (선택)

카메라가 삭제되면:

```text
cam1/
```

빈 폴더만 남을 수 있습니다.

선택적으로:

```python
parent = oldest.parent

oldest.unlink()

if not any(parent.iterdir()):
    parent.rmdir()
```

를 추가할 수 있습니다.

---

### 최종 수정본

```python
def purge_old_files_until(
    video_path: str,
    target_percent: float,
    extensions: list[str] = [".ts"],
):
    camera_files = defaultdict(list)

    files = []

    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))

    files = [f for f in files if f.is_file()]

    for f in files:
        camera = f.parent.name
        camera_files[camera].append(f)

    for camera in camera_files:
        camera_files[camera].sort(key=recording_time)

    while disk_usage_percent(video_path) > target_percent:

        deleted = False

        for _, file_list in camera_files.items():

            if not file_list:
                continue

            oldest = file_list.pop(0)

            try:
                oldest.unlink()

                logger.warning(
                    "Deleted old recording: %s",
                    oldest,
                )

                deleted = True

            except Exception:
                logger.exception(
                    "Failed deleting %s",
                    oldest,
                )

        if not deleted:
            break
```

---

추가로 성능을 생각하면 `pop(0)`은 O(n)입니다.

카메라당 수천~수만 개 세그먼트가 쌓일 수 있다면:

```python
from collections import deque

camera_files[camera] = deque(
    sorted(camera_files[camera], key=recording_time)
)

oldest = camera_files[camera].popleft()
```

로 바꾸는 것이 더 좋습니다.

녹화 시스템이라면 장기적으로는 `deque + popleft()`를 추천합니다.

-------------------
## Question
```
이렇게 하면 된다는건가?

def purge_old_files_until(video_path: str, target_percent: float, extensions: list[str]=[".ts"]):
    camera_files = deque(list)
    files = []
    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))
    
    files = [f for f in files if f.is_file()]
    
    for f in files:
        camera = f.parent.name
        camera_files[camera].append(f)
    
    for camera in camera_files:
        camera_files[camera].sorted(camera_files[camera], key=recording_time)
    
    while disk_usage_percent(video_path=video_path) > target_percent:
        deleted = False
        
        for _, file_list in camera_files.items():
            if not file_list:
                continue
            
            oldest = file_list.pop(0)
            try:
                oldest.unlink()
                deleted=True
            except Exception:
                logger.exception("Failed Deleting %s", oldest)
                
        if not deleted:
            break
```
-----------------------
## Answer
아직 몇 군데 문법/자료구조가 섞여 있습니다.

### 문제점 1

```python
camera_files = deque(list)
```

이건 안 됩니다.

원하시는 건:

```python
camera_files[camera] = deque([...])
```

형태의 "카메라별 deque"입니다.

따라서:

```python
from collections import defaultdict, deque

camera_files = defaultdict(deque)
```

로 시작해야 합니다.

---

### 문제점 2

```python
camera_files[camera].sorted(...)
```

이런 메서드는 없습니다.

`sorted()`는 내장 함수입니다.

```python
camera_files[camera] = deque(
    sorted(
        camera_files[camera],
        key=recording_time,
    )
)
```

처럼 써야 합니다.

---

### 문제점 3

`deque`를 쓰는데

```python
oldest = file_list.pop(0)
```

를 쓰고 있습니다.

`deque`는:

```python
file_list.popleft()
```

를 사용해야 O(1)입니다.

```python
oldest = file_list.popleft()
```

---

### 수정된 버전

```python
from collections import defaultdict, deque
from pathlib import Path


def purge_old_files_until(
    video_path: str,
    target_percent: float,
    extensions: list[str] = [".ts"],
):
    camera_files = defaultdict(deque)

    files = []

    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))

    files = [f for f in files if f.is_file()]

    for f in files:
        camera = f.parent.name
        camera_files[camera].append(f)

    for camera in camera_files:
        camera_files[camera] = deque(
            sorted(
                camera_files[camera],
                key=recording_time,
            )
        )

    while disk_usage_percent(video_path) > target_percent:

        deleted = False

        for camera, file_list in camera_files.items():

            if not file_list:
                continue

            oldest = file_list.popleft()

            try:
                oldest.unlink()

                logger.warning(
                    "Deleted %s (%s)",
                    camera,
                    oldest.name,
                )

                deleted = True

            except Exception:
                logger.exception(
                    "Failed deleting %s",
                    oldest,
                )

        if not deleted:
            break
```

---

그런데 한 가지 더 생각해볼 부분이 있습니다.

현재 구현은:

```text
1차 삭제
cam1 1개
cam2 1개
cam3 1개

2차 삭제
cam1 1개
cam2 1개
cam3 1개
```

즉 **카메라별 균등 삭제**입니다.

만약:

```text
cam1 : 1000개
cam2 : 10개
cam3 : 5개
```

라면 얼마 지나지 않아 cam2, cam3는 영상이 하나도 안 남고 cam1만 많이 남게 됩니다.

실제 NVR/CCTV는 보통:

```python
files.sort(key=recording_time)
```

후

```python
가장 오래된 파일 하나씩
```

전역 삭제를 더 많이 사용합니다.

그래서 정말 "카메라마다 하나씩 삭제"가 필요한지, 아니면 "전체 영상 중 가장 오래된 것부터 삭제"가 필요한지 한 번 더 확인해 보는 것이 좋습니다. 두 정책은 운영 결과가 꽤 다릅니다.
----------------------
## Question
```
됐나?

def purge_old_files_until(video_path: str, target_percent: float, extensions: list[str]=[".ts"]):
    camera_files = defaultdict(deque)
    files = []
    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))
    
    files = [f for f in files if f.is_file()]
    
    for f in files:
        camera = f.parent.name
        camera_files[camera].append(f)
    
    for camera in camera_files:
        camera_files[camera] = deque(sorted(camera_files[camera], key=recording_time))
    
    while disk_usage_percent(video_path=video_path) > target_percent:
        deleted = False
        
        for _, file_list in camera_files.items():
            if not file_list:
                continue
            
            oldest = file_list.popleft()
            try:
                oldest.unlink()
                deleted=True
            except Exception:
                logger.exception("Failed Deleting %s", oldest)
                
        if not deleted:
            break
```
## Answer
네, 문법적으로는 거의 완성됐습니다.

다만 실서비스에 넣기 전에 2가지만 추가하는 것을 권장합니다.

### 1. 삭제 전에 파일 존재 여부 확인

녹화 스레드나 다른 프로세스가 파일을 건드릴 가능성이 있다면:

```python
try:
    if oldest.exists():
        oldest.unlink()
        deleted = True
except Exception:
    logger.exception("Failed Deleting %s", oldest)
```

---

### 2. 삭제 로그 남기기

나중에 디스크가 부족해졌을 때 원인 파악이 쉬워집니다.

```python
try:
    size_mb = oldest.stat().st_size / (1024 * 1024)

    oldest.unlink()

    logger.warning(
        "Deleted recording: %s (%.1f MB)",
        oldest,
        size_mb,
    )

    deleted = True

except Exception:
    logger.exception(
        "Failed Deleting %s",
        oldest,
    )
```

---

### 3. (선택) 루프 중간에 사용률 재확인

현재 구현은

```text
cam1 삭제
cam2 삭제
cam3 삭제
```

를 한 세트로 수행합니다.

만약 카메라가 100대이고 파일 하나가 500MB라면:

```text
삭제 목표: 85%

현재: 90.1%

cam1 삭제 → 89.8%
cam2 삭제 → 89.5%
...
cam20 삭제 → 84.9%
```

이미 목표에 도달했는데도 계속 삭제하게 됩니다.

이를 막으려면:

```python
for _, file_list in camera_files.items():

    if disk_usage_percent(video_path) <= target_percent:
        break

    ...
```

를 넣을 수 있습니다.

---

그래서 최종적으로는 이런 형태가 가장 안전합니다.

```python
while disk_usage_percent(video_path) > target_percent:

    deleted = False

    for _, file_list in camera_files.items():

        if disk_usage_percent(video_path) <= target_percent:
            break

        if not file_list:
            continue

        oldest = file_list.popleft()

        try:
            size_mb = oldest.stat().st_size / (1024 * 1024)

            oldest.unlink()

            logger.warning(
                "Deleted recording: %s (%.1f MB)",
                oldest,
                size_mb,
            )

            deleted = True

        except Exception:
            logger.exception(
                "Failed Deleting %s",
                oldest,
            )

    if not deleted:
        break
```

이 버전이면:

* 카메라별 균등 삭제
* 오래된 순서 삭제
* 목표 사용률 도달 시 즉시 중단
* 삭제 로그 기록

까지 모두 처리됩니다.


----------
## Question
```
최종적으로, main 에 호출할 storage_manager 를 위한 codeSet  은 이정도면 되나?
cam1, cam2, cam3 를 한 세트로 수행하는 것은 의도된 사항임.

from pathlib import Path
from collections import defaultdict, deque
from datetime import datetime
import time
import shutil
import logging

logger = logging.getLogger()

def disk_usage_percent(video_path: str):
    usage = shutil.disk_usage(video_path)
    return (usage.used / usage.total) * 100.0

def recording_time(path: Path):
    return datetime.strptime(path.stem, "%Y%m%d_%H%M%S")

def purge_old_files_until(video_path: str, target_percent: float, extensions: list[str]=[".ts"]):
    camera_files = defaultdict(deque)
    files = []
    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))
    
    files = [f for f in files if f.is_file()]
    
    for f in files:
        camera = f.parent.name
        camera_files[camera].append(f)
    
    for camera in camera_files:
        camera_files[camera] = deque(sorted(camera_files[camera], key=recording_time))
    
    while disk_usage_percent(video_path=video_path) > target_percent:
        deleted = False
        
        for _, file_list in camera_files.items():
            if not file_list:
                continue
            
            oldest = file_list.popleft()
            try:
                size_mb = oldest.stat().st_size / (1024 * 1024)
                oldest.unlink()
                logger.warning("Deleted Recording: %s (%.1f MB)", oldest, size_mb)
                deleted=True
            except Exception:
                logger.exception("Failed Deleting %s", oldest)
                
        if not deleted:
            break

    
def storage_manager(video_path: str, ext: list=[".ts"]):
    while True:
        if disk_usage_percent(video_path)> 90:
            purge_old_files_until(video_path=video_path, target_percent=80.0, extensions=ext)
        time.sleep(60)
```
## Answer
네. 전체적으로는 의도대로 동작합니다.

특히 사용자가 의도한:

```text
1차 삭제
cam1 oldest
cam2 oldest
cam3 oldest

2차 삭제
cam1 second
cam2 second
cam3 second
```

방식은 현재 코드로 구현되어 있습니다.

다만 실서비스 투입 전 4가지는 수정하는 것을 권장합니다.

---

### 1. mutable default argument

현재:

```python
def purge_old_files_until(
    video_path: str,
    target_percent: float,
    extensions: list[str] = [".ts"]
):
```

```python
def storage_manager(
    video_path: str,
    ext: list = [".ts"]
):
```

Python에서는 기본 인자로 리스트를 두는 것을 피하는 것이 좋습니다.

```python
def purge_old_files_until(
    video_path: str,
    target_percent: float,
    extensions: list[str] | None = None,
):
    if extensions is None:
        extensions = [".ts"]
```

```python
def storage_manager(
    video_path: str,
    ext: list[str] | None = None,
):
    if ext is None:
        ext = [".ts"]
```

---

### 2. storage_manager 예외 처리

현재는 예외 발생 시 스레드가 죽습니다.

예:

```python
oldest.stat()
```

에서 에러 발생

↓

```python
storage_manager 종료
```

그래서:

```python
def storage_manager(video_path: str, ext=None):
    if ext is None:
        ext = [".ts"]

    while True:
        try:
            usage = disk_usage_percent(video_path)

            if usage > 90:
                purge_old_files_until(
                    video_path=video_path,
                    target_percent=80.0,
                    extensions=ext,
                )

        except Exception:
            logger.exception("Storage Manager Error")

        time.sleep(60)
```

를 추천합니다.

---

### 3. recording_time 실패 대응

현재:

```python
datetime.strptime(
    path.stem,
    "%Y%m%d_%H%M%S"
)
```

파일명이 하나라도 이상하면 전체 정렬이 실패합니다.

예:

```text
cam1/current.ts
cam1/temp.ts
```

같은 파일이 생기면 예외 발생.

안전하게:

```python
def recording_time(path: Path):
    try:
        return datetime.strptime(
            path.stem,
            "%Y%m%d_%H%M%S",
        )
    except ValueError:
        return datetime.max
```

를 추천합니다.

그러면 이상한 파일은 맨 뒤로 밀립니다.

---

### 4. 삭제 후 usage 재계산 로그

운영 시 매우 유용합니다.

```python
logger.warning(
    "Deleted Recording: %s (%.1f MB)",
    oldest,
    size_mb,
)
```

뒤에:

```python
logger.info(
    "Current Disk Usage: %.2f%%",
    disk_usage_percent(video_path),
)
```

정도 추가하면 디버깅이 쉬워집니다.

---

최종 평가:

✅ 삭제 정책 정상

✅ 카메라별 균등 삭제 정상

✅ 90% → 80% 히스테리시스 정상

✅ 무한루프 방지 정상

⚠️ 예외 처리 추가 권장

⚠️ mutable default 수정 권장

현재 상태도 동작은 하겠지만, 위 두 가지(기본 인자, storage_manager 예외 처리)는 거의 필수 수준으로 넣는 것을 추천합니다.

---------------
## Question
```
from pathlib import Path
from collections import defaultdict, deque
from datetime import datetime
import time
import shutil
import logging

logger = logging.getLogger()

def disk_usage_percent(video_path: str):
    usage = shutil.disk_usage(video_path)
    return (usage.used / usage.total) * 100.0

def recording_time(path: Path):
    try:
        return datetime.strptime(path.stem, "%Y%m%d_%H%M%S")
    except ValueError:
        return datetime.max

def purge_old_files_until(video_path: str, target_percent: float, extensions: list[str] | None = None):
    if ext is None:
        ext = [ ".ts" ]
    camera_files = defaultdict(deque)
    files = []
    for ext in extensions:
        files.extend(Path(video_path).rglob(f"*{ext}"))
    
    files = [f for f in files if f.is_file()]
    
    for f in files:
        camera = f.parent.name
        camera_files[camera].append(f)
    
    for camera in camera_files:
        camera_files[camera] = deque(sorted(camera_files[camera], key=recording_time))
    
    while disk_usage_percent(video_path=video_path) > target_percent:
        deleted = False
        
        for _, file_list in camera_files.items():
            if not file_list:
                continue
            
            oldest = file_list.popleft()
            try:
                size_mb = oldest.stat().st_size / (1024 * 1024)
                oldest.unlink()
                logger.warning("Deleted Recording: %s (%.1f MB)", oldest, size_mb)
                logger.info("Current Disk Usage: %.2f%%", disk_usage_percent(video_path=video_path))
                deleted=True
            except Exception:
                logger.exception("Failed Deleting %s", oldest)
                
        if not deleted:
            break

    
def storage_manager(video_path: str, target_percent: float, ext: list[str] | None = None):
    if ext is None:
        ext = [ ".ts" ]
    while True:
        try:
            if disk_usage_percent(video_path) > target_percent:
                purge_old_files_until(video_path=video_path, target_percent=target_percent, extensions=ext)
        except Exception:
            logger.exception("Storage Manager Error")
            
        time.sleep(60)

이 정도면 되나?
```
## Answer
거의 다 됐는데, **치명적인 버그가 2개** 있습니다.

### 1. `ext` / `extensions` 변수명 오류

현재:

```python
def purge_old_files_until(
    video_path: str,
    target_percent: float,
    extensions: list[str] | None = None
):
    if ext is None:
        ext = [".ts"]
```

여기서 `ext`는 존재하지 않습니다.

수정:

```python
if extensions is None:
    extensions = [".ts"]
```

그리고 아래도:

```python
for ext in extensions:
```

그대로 사용.

---

### 2. target_percent 의미가 꼬여 있음

현재:

```python
def storage_manager(
    video_path: str,
    target_percent: float,
    ...
):
    if disk_usage_percent(video_path) > target_percent:
        purge_old_files_until(
            video_path=video_path,
            target_percent=target_percent,
        )
```

이렇게 되면:

```python
storage_manager(
    "/recordings",
    target_percent=90
)
```

일 때

```text
90% 초과
↓
90% 이하가 될 때까지 삭제
```

가 됩니다.

보통 원하는 건:

```text
90% 초과 시
↓
80%까지 삭제
```

즉 High Watermark와 Low Watermark를 분리하는 것입니다.

예:

```python
def storage_manager(
    video_path: str,
    high_watermark: float = 90.0,
    low_watermark: float = 80.0,
    ...
):
```

그리고:

```python
if disk_usage_percent(video_path) > high_watermark:
    purge_old_files_until(
        video_path=video_path,
        target_percent=low_watermark,
        extensions=ext,
    )
```

---

### 권장 최종 형태

```python
def storage_manager(
    video_path: str,
    high_watermark: float = 90.0,
    low_watermark: float = 80.0,
    ext: list[str] | None = None,
):
```

사용:

```python
threading.Thread(
    target=storage_manager,
    kwargs={
        "video_path": "/recordings",
        "high_watermark": 90.0,
        "low_watermark": 80.0,
    },
    daemon=True,
).start()
```

---

그 외에는 현재 코드가 의도대로 동작합니다.

삭제 순서는:

```text
cam1 oldest
cam2 oldest
cam3 oldest

cam1 second
cam2 second
cam3 second
...
```

으로 진행되고,

```text
디스크 사용률 > 90%
```

가 되면

```text
80%
```

까지 떨어질 때까지 계속 삭제하는 구조로 만들면 안정적으로 운영할 수 있습니다.
