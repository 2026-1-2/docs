# 2026. 05. 10. (일) (Powered by Gemini 3.1)
## Question
```
Docker 컨테이너 stop 을 하면 Permission Denied 가 뜨는데, 왜 그러지?

모든 컨테이너에서 Permission Denied 가 뜨네.

root 에서 해도 안됨.
```
## Answer
`root` 권한으로 시도해도 `Permission Denied`가 발생하는 것은 일반적인 파일 시스템 권한 문제가 아니라, **커널 수준의 보안 모듈**이나 **프로세스 상태의 강한 제약** 때문일 확률이 매우 높습니다.

가장 유력한 원인들과 해결 방법을 정리해 드립니다.

---

## 1. AppArmor 또는 SELinux 보안 정책 충돌 (가장 흔함)

운영체제(주로 Ubuntu의 AppArmor나 CentOS의 SELinux)가 Docker의 프로세스 제어를 차단하고 있을 수 있습니다. `root`라고 해도 보안 모듈이 거부하면 명령이 먹히지 않습니다.

* **해결 방법 (AppArmor 리로드):**
```bash
sudo systemctl reload apparmor
sudo systemctl restart docker

```


* **해결 방법 (SELinux 일시 정지):**

```bash
    sudo setenforce 0
    # 이후 docker stop 시도. 해결된다면 SELinux 정책 수정 필요.
    ```

## 2. 파일 시스템이 Read-Only로 변경됨
시스템 오류나 디스크 장애로 인해 Docker 저장소(`/var/lib/docker`)가 포함된 파일 시스템이 **읽기 전용(Read-Only)** 상태로 마운트되었을 수 있습니다. 컨테이너를 중지할 때 상태를 기록해야 하는데, 쓰기가 불가능하면 권한 에러가 뜹니다.

*   **확인 방법:**
    
```bash
    mount | grep ' / '  # 혹은 Docker가 설치된 파티션 확인
    ```
    출력 결과에 `ro` (Read-Only)가 포함되어 있다면 시스템을 재부팅하거나 `rw`로 다시 마운트해야 합니다.

## 3. 컨테이너가 '데드락' 또는 좀비 상태
컨테이너 내의 프로세스가 커널 작업(I/O 대기 등) 중에 멈춰버려, Docker 데몬이 해당 프로세스에 시그널을 보낼 수 없을 때 발생합니다.

*   **해결 방법 (프로세스 직접 강제 종료):**
    1. 컨테이너의 실제 PID를 찾습니다.
       
```bash
       docker inspect --format '{{.State.Pid}}' [컨테이너_ID]
       ```
    2. 해당 PID를 `kill` 명령어로 강제 종료합니다.
       
```bash
       sudo kill -9 [찾은_PID]
       ```

## 4. Docker 데몬 자체의 상태 꼬임
Docker 서비스 자체가 내부적으로 일시적인 오류를 일으켰을 수 있습니다. 이 경우 컨테이너 제어 명령이 모두 거부될 수 있습니다.

*   **해결 방법:**
    ```bash
    sudo systemctl restart docker
    ```
    *주의: 실행 중인 모든 컨테이너가 재시작될 수 있습니다.*

## 5. 로그 확인을 통한 정확한 원인 파악
위의 방법으로 해결되지 않는다면, 시스템 로그를 통해 **진짜 거부 사유**를 확인해야 합니다.

*   **커널 로그 확인:**
    ```bash
    dmesg | tail -n 20
    ```
    (여기서 `audit: type=1400 ... apparmor="DENIED"` 같은 메시지가 보인다면 AppArmor 문제입니다.)
*   **Docker 로그 확인:**
    ```bash
    journalctl -u docker.service -n 50
    ```

---

### 추천 해결 순서
1. `dmesg` 명령어로 **AppArmor**나 **I/O Error**가 찍히는지 먼저 확인하세요.
2. 만약 AppArmor 문제라면 **서비스 리로드**를, 디스크 문제라면 **재부팅**을 권장합니다.
3. 급하게 꺼야 한다면 **3번 방법(PID 직접 Kill)**이 가장 확실합니다.

```
----------------
## Question
```
[  453.685211] audit: type=1400 audit(1778405083.808:178): apparmor="DENIED" operation="signal" class="signal" profile="docker-default" pid=8782 comm="runc" requested_mask="receive" denied_mask="receive" signal=quit peer="snap.docker.dockerd"

apparmor 에 의해 차단되는건가?
```
## Answer
네, 정확합니다. 올려주신 커널 로그를 보면 **AppArmor가 `runc`와 `snap.docker.dockerd` 사이의 신호(Signal) 전달을 차단**하고 있는 것이 확인됩니다.

### 로그 분석 결과

* **`apparmor="DENIED"`**: AppArmor 보안 정책에 의해 거부됨.
* **`operation="signal"`**: 프로세스 간에 신호(Stop 요청 등)를 보내는 행위가 차단됨.
* **`signal=quit`**: 컨테이너를 종료하기 위해 보낸 `SIGQUIT` 시그널이 도달하지 못함.
* **`peer="snap.docker.dockerd"`**: 현재 **Snap**을 통해 설치된 Docker를 사용 중인데, 이 Snap의 격리 환경과 Docker의 기본 프로필(`docker-default`)이 서로 충돌하고 있습니다.

Snap으로 설치된 Docker는 보안상 매우 엄격한 샌드박스 내에서 작동하기 때문에, 때때로 컨테이너 제어 명령(stop, kill 등)이 커널 수준에서 거부되는 고질적인 문제가 있습니다.

---

### 해결 방법

#### 1. AppArmor 프로필 새로고침 (가장 빠른 시도)

Snap 버전 Docker의 보안 프로필을 다시 로드하여 일시적인 꼬임을 해결합니다.

```bash
sudo apparmor_parser -r /var/lib/snapd/apparmor/profiles/snap.docker.dockerd
sudo systemctl restart snap.docker.dockerd

```

#### 2. Snap 인터페이스 재연결

Docker 커넥션 상태를 다시 설정하여 권한을 복구합니다.

```bash
sudo snap connect docker:home
sudo snap connect docker:firewall-control
sudo snap connect docker:support

```

#### 3. (권장) Snap 버전 제거 및 공식 Docker CE 설치

연구나 개발 목적으로 시스템 깊숙한 곳을 건드려야 하거나(eBPF, 커널 모듈 활용 등), 컨테이너를 빈번하게 제어해야 한다면 **Snap 버전 Docker는 제약이 너무 많습니다.**

공식 저장소(`.deb` 패키지)를 통해 설치하는 것이 훨씬 안정적입니다.

**기존 Snap 삭제:**

```bash
sudo snap remove docker

```

**공식 버전 설치 (Ubuntu 기준 예시):**

1. Docker 공식 GPG 키 및 저장소 추가
2. `apt-get install docker-ce docker-ce-cli containerd.io` 실행

---

### 당장 컨테이너를 꺼야 한다면?

AppArmor가 시그널을 막고 있어 `docker stop`이 안 먹히는 상태이므로, 강제로 프로세스를 찾아 죽여야 합니다.

1. **컨테이너 PID 확인**
```bash
docker inspect --format '{{.State.Pid}}' [컨테이너_ID]

```


2. **SIGKILL 전달 (AppArmor도 이 강제 종료는 막지 못하는 경우가 많습니다)**

```bash
   sudo kill -9 [확인된_PID]

```

현재 상황은 Snap 패키징의 격리 정책과 Docker 컨테이너의 격리 정책이 충돌하는 전형적인 케이스이므로, 가급적 **공식 패키지 설치**를 검토해 보시는 것을 추천드립니다.

```

```
----------------
## Question
```
sudo snap remove docker

이거 snapshot 없이 하는 방법?
```
## Answer
Snap에서 삭제 시 스냅샷(데이터 백업)을 생성하지 않으려면 **`--purge`** 옵션을 사용하면 됩니다.

```bash
sudo snap remove --purge docker

```

### 상세 설명

* **`--purge`**: 이 옵션을 추가하면 사용자 데이터와 설정을 보관하는 자동 스냅샷 생성을 건너뛰고 바로 삭제합니다.
* 만약 이 옵션 없이 삭제하면 `/var/lib/snapd/snapshots/` 경로에 데이터가 압축되어 저장되는데, 공식 버전(`apt`)으로 넘어가려는 상황이라면 굳이 필요 없는 용량만 차지하게 됩니다.

---

### 깔끔한 삭제를 위한 추가 팁

Snap을 지운 후에도 찌꺼기 파일이 남아있을 수 있습니다. 공식 버전을 설치하기 전에 아래 디렉토리들도 체크해서 비워주면 충돌 방지에 도움이 됩니다.

1. **잔여 디렉토리 삭제:**
```bash
sudo rm -rf /var/lib/docker
sudo rm -rf /etc/docker
sudo groupdel docker # (필요시)

```


2. **환경 변수/경로 확인:**
`which docker`를 입력했을 때 아무것도 나오지 않아야 정상입니다. 만약 `/snap/bin/docker` 등이 여전히 잡힌다면 터미널을 재시작하거나 `hash -r` 명령어로 경로 캐시를 비워주세요.

이제 공식 `docker-ce` 설치로 넘어가시면 AppArmor 시그널 차단 문제에서 훨씬 자유로워지실 겁니다.

```

```

----------------
## Question
```
May 10 18:28:50 hcseo systemd[1]: docker.service: Scheduled restart job, restart counter is at 3.
May 10 18:28:50 hcseo systemd[1]: docker.service: Start request repeated too quickly.
May 10 18:28:50 hcseo systemd[1]: docker.service: Failed with result 'exit-code'.
May 10 18:28:50 hcseo systemd[1]: Failed to start docker.service - Docker Application Container Engine.
May 10 18:29:25 hcseo systemd[1]: docker.service: Start request repeated too quickly.
May 10 18:29:25 hcseo systemd[1]: docker.service: Failed with result 'exit-code'.
May 10 18:29:25 hcseo systemd[1]: Failed to start docker.service - Docker Application Container Engine.
May 10 18:29:31 hcseo systemd[1]: docker.service: Start request repeated too quickly.
May 10 18:29:31 hcseo systemd[1]: docker.service: Failed with result 'exit-code'.
May 10 18:29:31 hcseo systemd[1]: Failed to start docker.service - Docker Application Container Engine.


왜 이러지?
```
## Answer
`Start request repeated too quickly`는 짧은 시간 안에 Docker가 반복적으로 재시작을 시도하다가 실패해서, systemd가 시스템 부하를 막으려고 잠시 시작을 차단한 상태입니다.

진짜 원인은 그 위에 찍혀 있을 `exit-code`가 왜 발생했는가에 있습니다. Snap 버전을 지우고 새로 설치하는 과정에서 설정이나 소켓이 꼬였을 가능성이 큽니다.

---

### 1. 진짜 원인 파악하기

systemd의 차단 제한을 무시하고, Docker가 왜 죽는지 상세 로그를 확인해야 합니다.

```bash
sudo journalctl -xeu docker.service

```

위 명령어를 입력했을 때, `failed to start...` 바로 직전 줄에 나오는 **Error 관련 메시지**(예: `failed to load config`, `address already in use`, `driver not supported`)를 찾는 것이 핵심입니다.

### 2. 가장 유력한 원인들과 해결책

#### **A. 기존 Docker 소켓과의 충돌**

Snap용 Docker가 사용하던 소켓 유령이 남아있어 새 Docker 서비스와 충돌할 수 있습니다.

```bash
sudo systemctl stop docker.socket
sudo rm -f /var/run/docker.sock
sudo systemctl start docker

```

#### **B. `/var/lib/docker` 데이터 충돌**

Snap 버전 Docker가 사용하던 데이터 구조와 현재 설치된 Docker 버전의 구조가 맞지 않으면 실행이 안 됩니다. 중요한 데이터가 없다면 이 디렉토리를 정리하는 것이 가장 확실합니다.

```bash
# 주의: 모든 이미지와 컨테이너 데이터가 삭제됩니다.
sudo rm -rf /var/lib/docker
sudo systemctl start docker

```

#### **C. `containerd` 상태 확인**

Docker의 하부 엔진인 `containerd`가 먼저 정상 작동해야 합니다.

```bash
sudo systemctl restart containerd
sudo systemctl status containerd

```

만약 `containerd`조차 `failed` 상태라면, Docker는 절대 켜지지 않습니다.

#### **D. `/etc/docker/daemon.json` 설정 오류**

혹시 설정을 위해 이 파일을 수정하셨다면, JSON 문법 오류나 지원하지 않는 옵션 때문에 실행이 거부될 수 있습니다. 파일이 있다면 잠시 이름을 바꿔서 테스트해 보세요.

```bash
sudo mv /etc/docker/daemon.json /etc/docker/daemon.json.bak
sudo systemctl start docker

```

---

### 3. 깨끗하게 재시도하는 법 (최후의 수단)

만약 계속 안 된다면, 설정과 소켓을 모두 초기화하고 다시 띄우는 것이 빠릅니다.

1. **서비스 중지:** `sudo systemctl stop docker docker.socket containerd`
2. **잔여물 제거:** `sudo rm -rf /var/lib/docker /var/run/docker.sock`
3. **서비스 시작:** `sudo systemctl start docker`

`journalctl -xeu docker.service`를 입력했을 때 나오는 **구체적인 에러 문구**(예: `failed to create containerd task` 등)를 알려주시면 더 정확한 원인을 짚어드릴 수 있습니다.

-----
----------------
## Question
```
ENTRYPOINT ["tini", "--"]

Python 프로그램을 도커 이미지로 만들었는데, 이거 쓰는게 좋나?
```
## Answer
결론부터 말씀드리면, **네, 아주 좋습니다.** 특히 Python 프로그램을 도커화할 때 `tini` 같은 **init 프로세스**를 사용하는 것은 단순한 권장 사항을 넘어 "베스트 프랙티스"로 통합니다.

왜 `tini`를 쓰는 것이 유리한지 시스템적인 관점에서 핵심 이유 2가지를 정리해 드립니다.

---

### 1. 시그널 전달 (Signal Forwarding)

도커 컨테이너가 시작되면 `ENTRYPOINT`에 지정된 프로세스는 리눅스 시스템의 **PID 1**을 부여받습니다. 리눅스에서 PID 1은 아주 특별한 역할을 하는데, 일반적인 프로세스와 달리 **커널이 기본적으로 제공하는 시그널 핸들러를 갖지 않습니다.**

* **문제점:** Python 인터프리터는 자기가 PID 1로 동작할 때 `SIGTERM`(종료 신호)을 받아도 이를 무시하거나 하위 프로세스에 전달하지 못하는 경우가 많습니다.
* **결과:** `docker stop`을 실행하면 Python이 신호를 무시하고, 도커는 10초 동안 기다리다가 결국 커널을 통해 `SIGKILL`로 강제 종료시킵니다. 이 과정에서 DB 연결 종료나 파일 쓰기 같은 **Clean-up 작업이 생략**될 수 있습니다.
* **tini의 역할:** `tini`가 PID 1이 되어 시그널을 먼저 받고, 이를 Python 프로그램에 확실하게 전달해 줍니다. 덕분에 프로그램이 안전하게 종료될 시간을 벌 수 있습니다.

### 2. 좀비 프로세스 청소 (Zombie Reaping)

리눅스 시스템에서 프로세스가 종료되면 부모 프로세스가 그 상태 코드를 읽어줘야 완전히 소멸됩니다. 만약 부모가 먼저 죽으면 해당 프로세스는 '고아(Orphan)'가 되어 PID 1(init)에게 입양됩니다.

* **문제점:** Python 애플리케이션 안에서 `subprocess` 등을 사용해 자식 프로세스를 생성했는데, 모종의 이유로 이들이 고아가 되면 PID 1인 Python 인터프리터가 이들을 "수거(wait)"해줘야 합니다. 하지만 일반적인 앱은 이런 설계가 되어 있지 않습니다.
* **결과:** 종료되지 못한 프로세스들이 **좀비(Zombie)** 상태로 시스템에 계속 남아 PID 자원을 낭비하게 됩니다.
* **tini의 역할:** `tini`는 입양된 고아 프로세스들이 종료될 때 즉시 상태 코드를 확인하여 시스템에서 완전히 제거해 줍니다.

---

### 어떻게 쓰는 게 가장 좋나?

작성하신 것처럼 Dockerfile에 포함하는 방식도 좋고, 도커 실행 시점에 주입할 수도 있습니다.

1. **Dockerfile 방식 (작성하신 코드):**
이미지 자체에 포함되어 어디서든 안정적으로 동작합니다.
```dockerfile
ENTRYPOINT ["tini", "--", "python", "app.py"]

```


2. **Docker 실행 플래그 방식:**
이미지에 `tini`가 없더라도 도커 엔진이 제공하는 기능을 쓸 수 있습니다.

```bash
    docker run --init my-python-app
    ```
    이 플래그를 쓰면 도커가 내부적으로 아주 가벼운 init 프로세스(사실 이 역시 tini 기반입니다)를 PID 1로 세워줍니다.

### 요약하자면
이전에 겪으셨던 **Permission Denied**나 **AppArmor** 문제와 직접적인 연관은 없지만, **컨테이너의 안정적인 종료와 자원 관리** 측면에서는 Python 개발 시 `tini`를 사용하는 것이 훨씬 견고한 구조를 만들어 줍니다. 특히 실서비스 환경이나 Kubernetes 환경으로 넘어갈 계획이 있다면 강력히 추천합니다.

```