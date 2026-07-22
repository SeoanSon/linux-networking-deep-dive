# 02: Linux Network Namespace 이론

## 🎯 핵심 질문

- **"왜" 네트워크를 격리해야 할까?** (문제 정의)
- 각 프로세스가 같은 포트에서 서비스할 수 있을까?
- 컨테이너들이 어떻게 독립적인 IP를 가질까?
- 하나의 서버에서 여러 개의 "서버"를 실행할 수 있을까?

---

## 📖 1. Network Namespace란?

### 정의
**Network Namespace**는 **격리된 네트워크 스택**입니다.

```
각 Network Namespace는 자신만의:
✅ IP 주소들 (eth0에 10.0.0.5, 또는 lo에 127.0.0.1)
✅ 포트들 (Port 80, 443, 8080 등 독립적 사용)
✅ 라우팅 테이블 (자신만의 경로 결정)
✅ 방화벽 규칙 (iptables, netfilter)
✅ 네트워크 디바이스 (eth0, lo, veth 등)
```

### 현실의 비유

```
🏢 회사 건물 (Host 시스템)
├─ 🚪 1층 사무실 (Namespace 1) - 인터넷: 10.0.0.5
├─ 🚪 2층 사무실 (Namespace 2) - 인터넷: 10.0.0.6  
├─ 🚪 3층 사무실 (Namespace 3) - 인터넷: 10.0.0.7
└─ 🚪 지하층 공용망 (Host Namespace) - 인터넷 직접 연결

각 층은:
- 자신만의 전화망 (포트)을 가짐
- 다른 층의 전화번호와 충돌 안 함
- 필요하면 중앙 교환대(라우팅)을 통해 통신
```

---

## 📖 2. Network Namespace의 역사

### Linux 개발 타임라인

```
2006년 ┌─ 초기 Namespace 구현 시작
       │  - PID Namespace (Eric W. Biederman)
       │  - 프로세스 격리 아이디어

2007년 │  - UTS Namespace (호스트명 격리)
       │  - IPC Namespace (프로세스 간 통신)
       ├─ Mount Namespace (파일시스템)

2009년 │  - Network Namespace (2.6.24 커널)
       │  - "진짜" 격리된 네트워킹
       │  - 컨테이너의 기초 탄생!
       ├─ User Namespace (2.6.23)

2013년 │  - Docker 등장
       │  - Network Namespace + cgroup 조합
       │  - 실제 컨테이너 실행 시작

2014년 │  - Kubernetes 탄생
       └─ Network Namespace의 진정한 가치 발휘
```

**핵심 시점**: Network Namespace 없으면 **Docker/Kubernetes 불가능**

---

## 📖 3. Network Namespace의 필요성

### 문제 1: 포트 충돌

```
Host 시스템 (Network Namespace 없음):
┌─────────────────────────┐
│ Process A (Nginx)       │
│ ├─ Port 80 ✅           │
│ ├─ IP 192.168.1.10      │
│                         │
│ Process B (MySQL)       │
│ ├─ Port 3306 ✅         │
│                         │
│ Process C (또다른 Nginx) │
│ ├─ Port 80 ❌ 이미 사용! │
│ └─ 실행 실패            │
└─────────────────────────┘
```

### 문제 2: IP 충돌

```
다른 팀의 서버들을 한 물리 서버에 배포하려고 함:

팀 A의 애플리케이션     팀 B의 애플리케이션
│                       │
├─ 내부 설정:          ├─ 내부 설정:
│  IP 10.0.0.1         │  IP 10.0.0.1 ← 같은 IP!
│  (팀만의 IP)         │  (팀만의 IP)
│
❌ 충돌! 어느 쪽으로 패킷이 갈까?
```

### 문제 3: 보안 (격리 없음)

```
Host 시스템:
Process A (신뢰할 수 있는 앱)   Process B (신뢰 불가 앱)
│                              │
└─────────────────────────────┘
      모두 같은 네트워크!
      
Process B가 Process A의 트래픽을 엿들을 수 있음 (Sniffing)
Process B가 전체 라우팅 규칙을 수정할 수 있음
Process B가 방화벽을 우회할 수 있음
```

### Network Namespace의 해결책

```
Network Namespace로 격리:
┌─────────────────────┐  ┌─────────────────────┐
│ Namespace 1         │  │ Namespace 2         │
│ ├─ Port 80: Nginx   │  │ ├─ Port 80: Apache  │
│ ├─ IP: 10.0.0.1     │  │ ├─ IP: 10.0.0.1     │
│ └─ 자신만의 라우팅  │  │ └─ 자신만의 라우팅  │
└─────────────────────┘  └─────────────────────┘
         ↓                        ↓
      충돌 없음!            각각 독립적으로 작동
```

---

## 📖 4. Network Namespace의 구성 요소

### 각 Namespace가 격리하는 것들

#### 4.1 네트워크 디바이스 (Network Devices)

```
Host Namespace:
┌─────────────────────────────────┐
│ $ ip link show                  │
│                                 │
│ 1: lo (loopback)                │
│    - 127.0.0.1 (자신과 통신)    │
│                                 │
│ 2: eth0 (external)              │
│    - 192.168.1.10 (인터넷)      │
│                                 │
│ 3: docker0 (bridge)             │
│    - 172.17.0.1 (컨테이너 관리) │
└─────────────────────────────────┘

Container Namespace (별도):
┌─────────────────────────────────┐
│ $ ip link show                  │
│                                 │
│ 1: lo (loopback - 다른 것)      │
│    - 127.0.0.1 (자신과 통신)    │
│                                 │
│ 2: eth0 (container용)           │
│    - 172.17.0.2 (Host가 준 IP)  │
│                                 │
│ (docker0는 보이지 않음)         │
└─────────────────────────────────┘
```

#### 4.2 IP 주소 (IP Addresses)

```
Host Namespace:
eth0: 192.168.1.10  ← 인터넷 IP
eth0: 172.17.0.1    ← 컨테이너 관리 IP
lo:   127.0.0.1     ← 로컬호스트

Container Namespace 1:
eth0: 172.17.0.2    ← 이 컨테이너만의 IP
lo:   127.0.0.1     ← 로컬호스트 (별개)

Container Namespace 2:
eth0: 172.17.0.3    ← 이 컨테이너만의 IP
lo:   127.0.0.1     ← 로컬호스트 (별개)
```

#### 4.3 라우팅 테이블 (Routing Table)

```
Host Namespace:
$ ip route show
192.168.1.0/24 dev eth0  # 직접 연결된 네트워크
172.17.0.0/16 dev docker0  # 컨테이너 네트워크
0.0.0.0/0 via 192.168.1.1 dev eth0 # 나머지는 게이트웨이로

Container Namespace:
$ ip route show
172.17.0.0/16 dev eth0  # 직접 연결 (컨테이너 네트워크만)
0.0.0.0/0 via 172.17.0.1 dev eth0 # 기본 게이트웨이는 Host

✅ 각 Namespace는 자신의 관점에서만 네트워크를 본다
```

#### 4.4 포트와 소켓 (Ports & Sockets)

```
Host Namespace:
프로세스 A (Nginx)
├─ TCP:80 ✅ 사용 가능
└─ TCP:443 ✅ 사용 가능

프로세스 B (MySQL)
├─ TCP:3306 ✅ 사용 가능

Container Namespace 1:
프로세스 C (Nginx in Container)
├─ TCP:80 ✅ 이것도 가능! (다른 공간)
├─ 실제로는 Host의 TCP:8080으로 매핑됨
└─ 하지만 Container 안에서는 80으로 보임

Container Namespace 2:
프로세스 D (또 다른 Nginx)
├─ TCP:80 ✅ 이것도 가능!
├─ 실제로는 Host의 TCP:8081로 매핑됨
└─ Container 안에서는 80으로 보임
```

#### 4.5 방화벽 규칙 (iptables)

```
Host의 iptables:
$ sudo iptables -L
- 모든 트래픽에 대한 필터링
- 포트포워딩 (8080→80)
- NAT 규칙

Container의 iptables:
$ sudo iptables -L
- (Host의 규칙이 보이지 않음)
- Container만의 규칙 설정 가능
- 격리된 필터링
```

---

## 📖 5. Network Namespace의 작동 원리

### 5.1 Namespace 생성 프로세스

```
1️⃣ Clone System Call
   ┌─────────────────────────────────┐
   │ clone(CLONE_NEWNET)             │
   │                                 │
   │ 새로운 Network Namespace을      │
   │ 생성하고 Process가 그 안에서    │
   │ 실행됨                          │
   └─────────────────────────────────┘

2️⃣ 초기 상태
   ┌─────────────────────────────────┐
   │ 새 Namespace의 상태:            │
   │ ├─ lo (loopback) 만 존재        │
   │ ├─ 127.0.0.1 만 사용 가능       │
   │ ├─ 네트워크 연결 불가 상태      │
   │ └─ 외부와 통신 불가             │
   └─────────────────────────────────┘

3️⃣ 네트워크 연결 (나중에)
   ┌─────────────────────────────────┐
   │ Namespace에 veth 연결            │
   │ ├─ Host에서 veth pair 생성       │
   │ ├─ Namespace에 한쪽 끝 이동      │
   │ ├─ IP 주소 할당                  │
   │ └─ 이제 통신 가능                │
   └─────────────────────────────────┘
```

### 5.2 Linux 커널의 Namespace 관리

```
/proc 파일시스템에서 조회:
$ cat /proc/[PID]/ns/net
ns:[4026531940]  ← Namespace ID (같으면 같은 Namespace)

다른 Process들:
$ cat /proc/1/ns/net    # PID 1 (init)
ns:[4026531840]  ← Host Namespace

$ cat /proc/12345/ns/net  # Container Process
ns:[4026532509]  ← 다른 Namespace

🔍 같은 ID = 같은 Namespace 공유
   다른 ID = 격리된 Namespace
```

### 5.3 Namespace 변경

```
Process A가 다른 Namespace으로 이동:

현재:
Process A ──────→ Namespace 1 (ID: 1001)

setns() 시스템 콜 후:
Process A ──────→ Namespace 2 (ID: 2001) ✅ 이동 완료

$ nsenter --net=/var/run/netns/container bash
  └─ 기존 Namespace로 진입 가능
```

---

## 📖 6. Namespace와 Linux 커널

### 6.1 Namespace 구현 위치 (커널 코드)

```
Linux Kernel Source (~/linux/kernel/):
├─ namespace.c           # 기본 Namespace 처리
├─ nsproxy.c             # Namespace 프록시
├─ net/                  # Network Namespace
│  ├─ net_namespace.c    # 구현체
│  └─ ...
├─ ipc/
│  └─ namespace.c        # IPC Namespace
├─ uts_ns.c              # UTS Namespace
├─ pid_ns.c              # PID Namespace
└─ user_namespace.c      # User Namespace
```

### 6.2 Namespace 사이의 격리

```
Host 커널은 각 Namespace를 추적:

struct nsproxy {
    struct uts_namespace *uts_ns;
    struct ipc_namespace *ipc_ns;
    struct mnt_namespace *mnt_ns;
    struct pid_namespace *pid_ns_for_children;
    struct net *net_ns;  ← Network Namespace 포인터
    struct user_namespace *user_ns;
};

Process가 시스템 콜할 때:
1. 시스템 콜 발생
2. 커널이 Process의 nsproxy 조회
3. 현재 Namespace의 리소스에만 접근
4. 다른 Namespace 리소스는 보이지 않음
```

### 6.3 네트워킹 스택의 분리

```
Host 커널 메모리:
┌──────────────────────────┐
│ Network Stack (Host)     │
│ ├─ 라우팅 테이블         │
│ ├─ ARP 캐시              │
│ ├─ 열린 소켓들           │
│ ├─ 네트워크 디바이스들   │
│ └─ netfilter 규칙        │
└──────────────────────────┘
        ↑
   Process A (Host Namespace)
   ├─ IP: 192.168.1.10
   └─ Port 80 사용

┌──────────────────────────┐
│ Network Stack (NS 1)     │ ← 별개의 메모리
│ ├─ 라우팅 테이블 (분리됨)│
│ ├─ ARP 캐시 (분리됨)     │
│ ├─ 열린 소켓들 (분리됨)  │
│ ├─ 네트워크 디바이스     │
│ └─ netfilter 규칙 (분리됨)
└──────────────────────────┘
        ↑
   Process B (Container NS)
   ├─ IP: 172.17.0.2
   └─ Port 80 사용 ✅ 충돌 없음!
```

---

## 📖 7. Namespace와 AKS

### 7.1 AKS Pod의 Namespace 구조

```
AKS Node 위의 Pod:
┌──────────────────────────────────┐
│ AKS Node (Linux)                 │
│                                  │
│ ┌────────────────────────────┐  │
│ │ Pod A                      │  │
│ │ Namespace ID: 4026532509   │  │
│ │ ├─ eth0: 10.244.0.5        │  │
│ │ ├─ lo: 127.0.0.1           │  │
│ │ └─ Container Process       │  │
│ └────────────────────────────┘  │
│                                  │
│ ┌────────────────────────────┐  │
│ │ Pod B                      │  │
│ │ Namespace ID: 4026532510   │  │
│ │ ├─ eth0: 10.244.0.6        │  │
│ │ ├─ lo: 127.0.0.1           │  │
│ │ └─ Container Process       │  │
│ └────────────────────────────┘  │
│                                  │
│ Host Namespace ID: 4026531840    │
│ ├─ cbr0 (Container Bridge)       │
│ ├─ eth0: 172.16.0.5 (Node IP)   │
│ └─ kubelet 프로세스              │
│                                  │
└──────────────────────────────────┘
```

### 7.2 Pod 간 통신 흐름

```
Pod A (NS 1)에서 Pod B (NS 2)로 통신:

1️⃣ Pod A의 애플리케이션이 10.244.0.6으로 패킷 전송
   └─ Pod A의 Namespace 안에서만 작동

2️⃣ veth 인터페이스를 통해 Host로 나옴
   └─ Host의 cbr0 bridge로 도착

3️⃣ Host의 라우팅이 10.244.0.6이 Pod B임을 알고
   └─ Pod B의 veth로 패킷 전달

4️⃣ veth의 다른 끝이 Pod B의 Namespace로
   └─ Pod B의 eth0으로 도착

5️⃣ Pod B의 애플리케이션이 패킷 수신
   └─ Pod B의 Namespace 안에서만 작동
```

### 7.3 AKS에서 Namespace 확인

```bash
# Node에 접속했을 때:
$ kubectl debug node/aks-node-1 -it --image=ubuntu

# Host Namespace에서:
$ ip netns list
# (비어있음 - 일반적으로 Host 관리용 Namespace만 있음)

# Pod의 Namespace 조회:
$ docker ps | grep pod_name
CONTAINER ID    IMAGE
abc123def456    myimage

$ docker inspect abc123def456 | grep Pid
"Pid": 12345

$ cat /proc/12345/ns/net
ns:[4026532509]  ← Pod의 Namespace ID

# Host와 비교:
$ cat /proc/1/ns/net
ns:[4026531840]  ← Host Namespace (다름!)
```

---

## 📖 8. Network Namespace의 생명주기

```
생성 (Create):
    ↓
clone(CLONE_NEWNET)
    └─ 새 Namespace 생성
    └─ 초기 상태: lo만 존재

구성 (Setup):
    ↓
veth 추가, IP 할당
    └─ 네트워크 연결 가능 상태

사용 (Use):
    ↓
프로세스가 이 Namespace에서 실행
    └─ 네트워크 리소스 격리

삭제 (Delete):
    ↓
마지막 프로세스가 떠날 때 자동 삭제
    └─ Namespace 정리
    └─ 리소스 회수
```

---

## 📖 9. 다른 Namespace와의 관계

### Network Namespace ↔ PID Namespace

```
Host 시스템에서:
$ ps aux | grep nginx
root   1234  0.0  /usr/sbin/nginx

$ cat /proc/1234/ns/net
ns:[4026531840]  ← Host Namespace

$ cat /proc/1234/ns/pid
ns:[4026531837]  ← Host PID Namespace

─────────────────────

Container 내에서:
$ ps aux | grep nginx
root  1  0.0  /usr/sbin/nginx

$ cat /proc/1/ns/net
ns:[4026532509]  ← Container Namespace (다름!)

$ cat /proc/1/ns/pid
ns:[4026532508]  ← Container PID Namespace (다름!)

🔍 같은 Namespace 안에서는 PID 1부터 시작
   다른 프로세스들을 볼 수 없음
```

### Network Namespace ↔ Mount Namespace

```
Host Mount Namespace:
/etc/hostname → "aks-node-1"
/etc/resolv.conf → "168.63.129.16"

Container Mount Namespace:
/etc/hostname → "pod-name-xyz"
/etc/resolv.conf → Pod의 DNS 설정

🔍 같은 Network Namespace지만
   다른 Mount Namespace = 각각의 파일 시스템
```

---

## 📖 10. Network Namespace의 제한사항

### 10.1 격리의 한계

```
❌ 격리되지 않는 것들:
├─ 물리 네트워크 카드 (eth0 자체는 공유)
├─ Host의 커널 버전
├─ 시스템 전체 리소스
└─ Host 관리자의 감시 능력

✅ 격리되는 것들:
├─ IP 주소
├─ 포트
├─ 라우팅 테이블
├─ 방화벽 규칙
└─ 네트워크 인터페이스 이름
```

### 10.2 성능 고려사항

```
Namespace 사이의 통신 비용:

같은 Namespace (직접):
localhost:80 → Process A
성능: ⭐⭐⭐⭐⭐ 극도로 빠름

같은 호스트, 다른 Namespace (veth via bridge):
Pod A → veth → cbr0 → veth → Pod B
성능: ⭐⭐⭐⭐ 빠름 (약간의 오버헤드)

다른 호스트 Namespace (라우팅 거쳐서):
Pod A → veth → Node A 라우팅 → 네트워크 → Node B → Pod B
성능: ⭐⭐⭐ 보통 (네트워크 레이턴시)
```

---

## 📖 11. Network Namespace와 실전

### 11.1 언제 필요한가?

```
✅ Network Namespace가 필요한 경우:

1️⃣ 컨테이너 환경
   └─ Docker, Kubernetes 등

2️⃣ 여러 서비스를 한 호스트에 배포
   └─ 포트 충돌 방지

3️⃣ 보안 격리
   └─ 신뢰할 수 없는 프로세스 격리

4️⃣ 네트워크 테스트
   └─ 로컬에서 멀티 호스트 시뮬레이션

5️⃣ 마이크로서비스 아키텍처
   └─ 각 서비스는 자신의 네트워크 설정
```

### 11.2 AKS에서의 실제 사용

```
AKS Pod = Process + Network Namespace:

각 Pod은:
├─ 자신의 IP 주소 가짐 (10.244.x.x)
├─ 자신의 포트 공간 가짐 (80, 443 등)
├─ 자신의 로컬호스트 가짐 (127.0.0.1)
└─ 다른 Pod과 완전히 격리됨

결과:
✅ 같은 Node에 여러 Pod 배포 가능
✅ Pod 간 포트 충돌 없음
✅ 각 Pod은 자신만의 "작은 Linux 호스트"처럼 작동
```

---

## 📖 12. 요약: Network Namespace의 핵심

### 한 문장 정의
**Network Namespace는 격리된 네트워크 스택으로, 각 Namespace가 자신만의 IP, 포트, 라우팅 테이블, 방화벽을 가진다.**

### 세 가지 핵심 역할

```
1️⃣ 격리 (Isolation)
   └─ 프로세스들이 네트워크 리소스 공유 안 함
   └─ 포트 충돌 제거

2️⃣ 가상화 (Virtualization)
   └─ 각 Namespace는 자신만의 "네트워크 환경" 보유
   └─ 마치 별도의 서버처럼 작동

3️⃣ 보안 (Security)
   └─ Namespace 간의 통신은 명시적으로만 가능
   └─ 격리되지 않은 프로세스가 다른 것 엿볼 수 없음
```

### 다음 장으로의 연결

```
이제 이해한 것:
✅ Network Namespace의 필요성
✅ 각 Namespace가 독립적인 네트워크 스택을 가짐
✅ Kubernetes Pod = Process + Network Namespace

다음 장 (veth pairs):
→ "그럼 격리된 Namespace들이 어떻게 서로 통신할까?"
→ "Network Namespace 사이의 연결 구조"
```

---

## 🎓 핵심 개념 체크

이 장을 이해했으면 다음에 답할 수 있어야 합니다:

- [ ] Network Namespace란 무엇이고, 왜 필요한가?
- [ ] 포트 충돌이 Namespace로 어떻게 해결되는가?
- [ ] 같은 호스트의 두 Namespace가 어떻게 독립적일 수 있는가?
- [ ] `/proc/[PID]/ns/net`은 무엇을 나타내는가?
- [ ] AKS Pod이 어떻게 Network Namespace를 사용하는가?
- [ ] Pod A에서 Pod B로의 통신 경로는?
- [ ] Network Namespace의 생명주기는?
- [ ] 격리되는 것과 격리되지 않는 것은 무엇인가?

---

## 📚 더 알아보기

### 관련 Linux 명령어 (다음 Hands-on에서)

```bash
# Namespace 목록 보기
ip netns list

# 새 Namespace 생성
sudo ip netns add test-ns

# Namespace 내에서 명령어 실행
sudo ip netns exec test-ns ip link show

# Namespace 삭제
sudo ip netns delete test-ns

# Process의 Namespace 확인
cat /proc/[PID]/ns/net

# Namespace로 진입
sudo nsenter --net=/var/run/netns/test-ns bash
```

### 커널 소스 참고

```
Linux Kernel: net/net_namespace.c
주요 함수:
- copy_net_ns()     # Namespace 생성
- get_net()         # Namespace 참조
- put_net()         # Namespace 해제
- setup_net()       # Namespace 초기화
```

---

**다음 주제: veth pairs - Namespace 사이의 "가상 케이블"** 🔌
