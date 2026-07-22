# 01: Linux 프로세스와 IPC (Inter-Process Communication)

> **목표**: Linux 프로세스의 구조를 이해하고, 프로세스 간 통신이 어떻게 이루어지는지 배우기
>
> **학습 시간**: 1.5시간 (이론 30분 + 실습 1시간)
>
> **난이도**: ⭐ 기초

---

## 📍 이 챕터의 위치

```
┌─────────────────────────────────────────┐
│  01: Linux Process & IPC ← YOU ARE HERE │
│  (프로세스 격리와 통신의 기초)            │
└────────────────┬────────────────────────┘
                 │
                 ↓
         ┌────────────────┐
         │ 02: Namespace  │
         │ (격리 메커니즘) │
         └────────────────┘
                 │
                 ↓
           [네트워크 스택]
```

---

## 🎯 학습 목표

이 챕터를 끝내면 다음을 이해할 수 있습니다:

- ✅ 리눅스 프로세스의 구조 (PID, 부모-자식 관계)
- ✅ 프로세스가 격리되는 방식 (UID/GID, 메모리 공간)
- ✅ 프로세스 간 통신 5가지 방법 (IPC)
- ✅ 이들이 **Kubernetes 격리**와 어떻게 연결되는지

---

## 🔗 선수 지식

- ✅ Linux 기본 명령어 (ls, ps, cat 등)
- ✅ Linux 파일 시스템 개념
- ❌ 커널 프로그래밍 경험 (필요 없음)
- ❌ C 언어 (기본 이해만 필요)

---

## 📚 문서 읽기 순서

```
1️⃣ README.md 읽기 (지금 보는 파일)
   ↓
2️⃣ Theory.md 정독 (개념 배우기)
   ↓
3️⃣ Architecture.md 다이어그램 분석
   ↓
4️⃣ Hands-on.md 실습 (직접 실행)
   ↓
5️⃣ PacketFlow.md 복습
   ↓
6️⃣ Troubleshooting.md 참고
   ↓
7️⃣ Interview.md로 개념 확인
```

---

## ❓ 이 챕터 핵심 질문

1. **프로세스란 무엇인가?**
   - 실행 중인 프로그램의 인스턴스
   - 고유한 PID를 가진 격리된 실행 단위

2. **왜 프로세스가 격리되어야 하는가?**
   - 한 프로세스의 오류가 다른 프로세스에 영향을 주지 않도록
   - 보안: 권한이 없는 프로세스가 다른 프로세스 메모리 접근 불가

3. **그럼 프로세스 간에 어떻게 통신하나?**
   - 명시적으로 IPC 메커니즘을 사용
   - 파이프, 소켓, 메시지 큐, 공유 메모리, 세마포어

4. **이게 Kubernetes와 무슨 상관인가?**
   - 컨테이너는 프로세스들의 모음
   - Namespace는 프로세스 격리를 강화
   - CNI는 프로세스(Pod)들의 통신을 관리

---

## 🏗️ 아키텍처 미리보기

```
User Space
├─ 프로세스 A (PID 1000)
│  ├─ 메모리 (4GB)
│  ├─ 파일 디스크립터
│  └─ 스택/힙
│
├─ 프로세스 B (PID 1001)
│  └─ [완전히 격리된 메모리]
│
└─ 프로세스 C (PID 1002)

커널 (Kernel Space)
├─ Process Table
│  ├─ PID 1000 → /bin/bash
│  ├─ PID 1001 → /bin/python
│  └─ PID 1002 → /bin/nginx
│
├─ IPC 메커니즘
│  ├─ Pipes
│  ├─ Sockets
│  ├─ Message Queues
│  ├─ Shared Memory
│  └─ Semaphores
│
└─ 메모리 관리
   ├─ MMU (Memory Management Unit)
   ├─ Virtual Memory
   └─ Context Switching
```

---

## 📡 Hands-on 미리보기

실습에서 다음을 직접 해봅니다:

```bash
# 1️⃣ 프로세스 구조 이해
ps -ef
cat /proc/1/status

# 2️⃣ 부모-자식 프로세스 확인
bash
sleep 1000 &
ps -ef --forest

# 3️⃣ 프로세스 격리 확인 (메모리)
cat /proc/self/maps

# 4️⃣ IPC 실습 (파이프)
cat file.txt | grep "pattern" | wc -l

# 5️⃣ 프로세스 간 통신 (소켓)
# 서버 시작
nc -l 5000
# 다른 터미널에서
echo "Hello" | nc localhost 5000
```

---

## 🌉 Kubernetes와의 연결

**Container**는 본질적으로 **격리된 프로세스들의 모음**입니다:

```
Kubernetes Pod
└─ PID Namespace (격리됨)
   ├─ Process 1 (app, PID=1)
   ├─ Process 2 (logging sidecar, PID=5)
   └─ Process 3 (monitoring agent, PID=10)

호스트 커널에서의 실제 상태:
├─ Container 1의 Process 1 → 호스트에서 PID=5000
├─ Container 1의 Process 2 → 호스트에서 PID=5001
├─ Container 2의 Process 1 → 호스트에서 PID=6000
└─ Container 2의 Process 2 → 호스트에서 PID=6001
```

**격리**:
- 각 Pod은 자신의 **PID Namespace**을 가짐
- 따라서 Pod A에서는 Pod B의 프로세스를 볼 수 없음
- 하지만 **네트워크 Namespace**는 공유할 수 있음 (멀티컨테이너 Pod)

---

## 🏢 Azure AKS 관점

### AKS에서 프로세스를 어떻게 볼까?

```bash
# AKS 노드에 SSH 접속했을 때
ps aux | grep kubelet  # Kubernetes의 프로세스
ps aux | grep docker   # 또는 containerd (Container Runtime)

# 특정 Pod의 프로세스 확인
kubectl exec -it <pod-name> -- ps aux

# 노드에서 해당 Pod의 실제 PID 확인
# (고급) 보다 복잡한 격리 메커니즘이 적용됨
```

### 엔터프라이즈 시나리오

| 문제 | 원인 | 해결책 |
|------|-----|--------|
| Pod이 자꾸 죽음 | 프로세스 메모리 부족 | Resource Limits 설정 |
| Pod이 응답 없음 | Zombie 프로세스 | Init 프로세스 설정 (PID=1) |
| Pod 간 통신 안 됨 | IPC 네임스페이스 격리 | Pod IPC mode 설정 |

---

## 💡 핵심 개념 미리보기

### 프로세스 생명주기

```
프로세스 생성 → 실행 중 → 종료
├─ fork()  │      │     │
├─ exec()  │      │     │
│          ├─ Running
│          ├─ Sleeping
│          ├─ Zombie
│          └─ Defunct
│                  │
└─────────────────→ exit()
                   └─ wait()로 부모가 처리
```

### IPC 5가지 방법

```
1️⃣ Pipe (|)
   - 단방향 통신
   - 예: cat file.txt | grep pattern

2️⃣ Named Pipe (FIFO)
   - 파일 시스템 기반
   - 예: mkfifo /tmp/mypipe

3️⃣ Socket
   - 양방향, 네트워크 지원
   - 예: nc (netcat)

4️⃣ Message Queue
   - 메시지 기반
   - 예: System V IPC

5️⃣ Shared Memory
   - 직접 메모리 공유
   - 가장 빠른 IPC (동기화 필요)
```

---

## 🎓 학습 포인트

### 이해해야 할 것

- [ ] PID와 프로세스의 관계
- [ ] 부모-자식 프로세스 계층
- [ ] 프로세스 메모리 공간 격리 (virtual memory)
- [ ] User/Group ID와 권한 (UID/GID)
- [ ] IPC의 5가지 종류와 사용 사례
- [ ] Zombie 프로세스란 무엇인가
- [ ] 프로세스 시그널 (SIGTERM, SIGKILL 등)

### 직접 해봐야 할 것

- [ ] ps 명령어로 프로세스 트리 보기
- [ ] /proc 파일 시스템으로 프로세스 분석
- [ ] 파이프로 프로세스 연결
- [ ] 소켓으로 프로세스 통신
- [ ] strace로 시스템 콜 추적
- [ ] 우발적 zombie 프로세스 생성

---

## 📊 이 챕터 로드맵

```
Theory.md
├─ Process 개념
├─ PID와 PPID
├─ 메모리 격리
├─ UID/GID
├─ IPC 5가지
└─ 시그널

Architecture.md
├─ 프로세스 구조도
├─ 메모리 레이아웃
├─ IPC 메커니즘 비교
└─ AKS에서의 모습

Hands-on.md
├─ 프로세스 확인
├─ 격리 테스트
├─ IPC 실습
└─ strace 사용

PacketFlow.md
└─ 프로세스 창조에서 종료까지 흐름

Troubleshooting.md
├─ Zombie 프로세스 해결
├─ 좀비 찾기
└─ 리소스 부족 대응

Interview.md
├─ "프로세스란?"
├─ "격리는 어떻게?"
├─ "IPC 방법은?"
└─ Kubernetes와의 연결
```

---

## 🚀 시작하기

### 최소 준비 (5분)

```bash
# 터미널 열기
# 다음 명령어 실행
ps aux
cat /proc/1/status
```

### 전체 실습 (1시간)

1. Theory.md 읽기 (20분)
2. Hands-on.md 따라하기 (40분)
3. 헷갈리는 부분 Troubleshooting.md 확인

---

## 📝 체크리스트

이 챕터 완료 기준:

- [ ] "프로세스"를 설명할 수 있다
- [ ] PID, PPID의 의미를 안다
- [ ] "격리"가 무엇인지 설명할 수 있다
- [ ] IPC 5가지를 나열할 수 있다
- [ ] ps, /proc을 사용할 수 있다
- [ ] 파이프와 소켓의 차이를 안다
- [ ] Zombie 프로세스 문제를 해결할 수 있다
- [ ] 고객에게 "왜 컨테이너는 격리되어야 하는가"를 설명할 수 있다

---

## ➡️ 다음으로

이 챕터를 완료했으면 **02-linux-network-namespace**로 진행하세요.

02에서는 프로세스 격리를 **네트워킹 관점**으로 확대합니다:

```
프로세스 격리 (이 챕터)
    ↓
네트워크 격리 (02장)
    ↓
가상 네트워크 장치 (03장)
    ↓
Linux 브릿지 (04장)
    ↓
... 최종적으로 Kubernetes 네트워킹!
```

---

**준비되셨나요? 👇**

```bash
cat Theory.md
```
