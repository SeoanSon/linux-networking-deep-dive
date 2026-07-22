# 04: Linux Bridge (L2 스위칭) 이론

## 🎯 핵심 질문

- **"여러 개의 veth pair를 어떻게 연결할까?"**
- Linux Bridge는 실제 스위치와 같은가?
- MAC 주소 기반 포워딩은 어떻게 작동하나?
- Kubernetes의 cbr0는 무엇인가?

---

## 📖 1. Linux Bridge란?

### 정의
**Linux Bridge**는 **여러 네트워크 인터페이스를 L2 (Data Link Layer)에서 연결하는 가상 스위치**입니다.

```
일반 물리 스위치:
┌──────────────────────┐
│ 스위치 (Switch)      │
├──────────────────────┤
│ Port 1  Port 2  Port 3│
└──────┬───────┬───────┘
       │       │
  ┌────┘       └────┐
┌─┴─┐           ┌──┴─┐
│PC1│           │PC2│
└───┘           └───┘

Linux Bridge (가상):
┌──────────────────────────┐
│ br0 (Linux Bridge)       │
├──────────────────────────┤
│ veth0  veth1  eth0       │
└──────┬───────┬────┬──────┘
       │       │    │
  ┌────┘       │    └────┐
  │        ┌───┘         │
┌─┴──┐  ┌──┴──┐    ┌─────┴──┐
│Pod A│  │Pod B│    │외부 네트│
└────┘  └─────┘    └────────┘
```

### 특징

```
✅ Linux Bridge의 특징:

1️⃣ L2 (MAC 기반) 스위칭
   └─ IP를 보지 않음, MAC만 봄
   └─ 패킷의 출발지/목적지 MAC을 확인

2️⃣ 여러 인터페이스 연결
   └─ br0에 veth0, veth1, eth0 등 추가
   └─ 모두가 같은 LAN 세그먼트처럼 작동

3️⃣ MAC 주소 학습
   └─ 패킷이 들어오면 어느 포트에서 왔는지 학습
   └─ 다음번엔 그 포트로만 전송

4️⃣ 투명성
   └─ 애플리케이션은 bridge의 존재를 모름
   └─ 일반 스위치처럼 작동

5️⃣ STP (Spanning Tree Protocol) 지원
   └─ 루프 방지 (선택사항)
```

---

## 📖 2. Linux Bridge의 역사

### 2.1 언제 등장했나?

```
2000년대 초 ┌─ Virtual networking 필요성 대두
            │  Linux VM들이 네트워킹 필요
            │
2003년 ├─ Xen hypervisor 개발 시작
       │  └─ Virtual bridging 필요

2004년 │  Linux 2.6 커널
       │  └─ net_bridge 모듈 도입
       │  └─ "진정한" 가상 브릿지

2006년 │  Namespace 개발과 함께
       │  └─ veth와 bridge의 조합
       │  └─ 컨테이너 기초 탄생

2013년 ├─ Docker 등장
       │  └─ docker0 bridge로 유명해짐

2015년 │  Kubernetes 등장
       │  └─ cbr0 (Container Bridge) 사용
       └─ L3 라우팅과 함께 사용
```

---

## 📖 3. Linux Bridge의 필요성

### 3.1 문제 상황

```
veth pair만으로는:
┌─ Namespace A    ┌─ Namespace B    ┌─ Namespace C
│ veth-A ←─veth─→ │ ???             │ ???
└────────────────┘└─────────────────┘└────────────

veth-B와 veth-C를 어떻게 연결하지?
각각 새로운 veth pair를 만들어야 함?

✅ veth A ↔ veth B (Namespace A ↔ B)
✅ veth C ↔ veth D (Namespace A ↔ C)
✅ veth E ↔ veth F (Namespace B ↔ C)

N개 Namespace = N*(N-1)/2 개 veth pair 필요!
복잡도: O(N²) → 매우 비효율적
```

### 3.2 Bridge의 해결책

```
Linux Bridge (br0)로 통합:

┌──────────────────────────────────────┐
│ Host Namespace                       │
│ ┌────────────────────────────────┐  │
│ │ br0 (Linux Bridge)             │  │
│ │ MAC: 02:42:ac:11:00:01        │  │
│ └────┬────────────┬────────┬────┘  │
│      │            │        │       │
│   veth-a      veth-b    veth-c    │
│      │            │        │       │
│      └────────────┼────────┘       │
│          ↑        ↑        ↑       │
└──────────┼────────┼────────┼────────┘
      ┌────┘   ┌────┘   ┌───┘
      │        │        │
┌─────┴──┐  ┌──┴──┐  ┌──┴───┐
│ Pod A  │  │Pod B│  │Pod C │
└────────┘  └─────┘  └──────┘

✅ 모두 같은 네트워크 (br0)에 연결
✅ N개 Pod = N개 veth pair만 필요
✅ 복잡도: O(N) → 매우 효율적
```

---

## 📖 4. Linux Bridge의 작동 원리

### 4.1 Bridge 생성과 포트 추가

```
1️⃣ Bridge 생성
   $ ip link add br0 type bridge
   
   결과:
   br0: <BROADCAST,MULTICAST>
   └─ MAC: 02:42:ac:11:00:01 (무작위)
   └─ 포트 없음 (비어있음)

2️⃣ 포트 추가
   $ ip link set veth-a master br0
   $ ip link set veth-b master br0
   $ ip link set veth-c master br0
   
   결과:
   br0에 3개 포트 추가됨

3️⃣ IP 할당 (선택사항)
   $ ip addr add 10.244.0.1/24 dev br0
   
   br0 자체도 IP를 가질 수 있음
   (이 경우 gateway 역할)
```

### 4.2 MAC 주소 학습 및 포워딩

```
초기 상태:
br0의 MAC 테이블:
(비어있음)

1️⃣ Pod A (veth-a)에서 Pod B (veth-b)로 패킷 전송:
   
   Ethernet Frame:
   ┌─────────────────────────────────┐
   │ Src MAC: 02:42:ac:11:00:02      │ (Pod A의 MAC)
   │ Dst MAC: 02:42:ac:11:00:03      │ (Pod B의 MAC)
   │ Data: ...                       │
   └─────────────────────────────────┘
   
   br0에 도착:
   └─ MAC 테이블에 학습: 02:42:ac:11:00:02 → veth-a

2️⃣ br0이 목적지 MAC 조회:
   
   MAC 테이블에 02:42:ac:11:00:03이 없음
   └─ Flooding: 모든 포트로 전송 (출발지 제외)
   └─ veth-a는 이미 출발지니까 제외
   └─ veth-b와 veth-c로 전송

3️⃣ Pod B가 응답 패킷 보냄:
   
   Ethernet Frame:
   ┌─────────────────────────────────┐
   │ Src MAC: 02:42:ac:11:00:03      │ (Pod B의 MAC)
   │ Dst MAC: 02:42:ac:11:00:02      │ (Pod A의 MAC)
   │ Data: ...                       │
   └─────────────────────────────────┘
   
   br0에 도착:
   └─ MAC 테이블에 학습: 02:42:ac:11:00:03 → veth-b

4️⃣ 다음 번 통신 (Pod A → Pod B):
   
   br0이 MAC 테이블 조회:
   └─ 02:42:ac:11:00:03 → veth-b (학습함)
   └─ veth-b로만 전송 (정확한 전달)
   └─ Flooding 불필요
```

### 4.3 커널 구조

```
Linux Kernel:

struct net_bridge {
    struct list_head port_list;    # 포트들의 리스트
    struct list_head hash[BR_HASH_SIZE];  # MAC 테이블
    struct br_mdb_entry mdb;       # 멀티캐스트
    ...
};

struct net_bridge_port {
    struct net_device *dev;        # 물리 디바이스
    struct list_head list;         # 포트 리스트
    struct net_bridge *br;         # 속한 bridge
    unsigned long path_cost;       # STP 비용
    ...
};

MAC 테이블 항목:
struct net_bridge_fdb_entry {
    struct hlist_node hlist;
    mac_addr addr;                 # 목적지 MAC
    unsigned char port_no;         # 어느 포트
    unsigned long ageing_time;     # 얼마나 오래 배웠나
};
```

---

## 📖 5. Linux Bridge와 IP

### 5.1 Bridge의 IP 역할

```
Linux Bridge는 L2 장치지만, IP도 가질 수 있음:

┌────────────────────────────────┐
│ br0 (10.244.0.1/24)           │
│ ├─ veth-a (Pod A)             │
│ ├─ veth-b (Pod B)             │
│ ├─ veth-c (Pod C)             │
│ └─ eth0 (외부 연결)           │
└────────────────────────────────┘

br0의 IP 의미:
├─ Gateway 역할
│  └─ Pod들의 기본 게이트웨이
│  └─ "이 IP로 보내면 br0이 처리"
│
└─ Host의 관리 IP
   └─ br0을 통한 트래픽 모니터링
   └─ 필요하면 Host에서 직접 통신

예: Pod A에서 10.244.0.0/24 밖으로 나갈 때
└─ default gateway: 10.244.0.1 (br0)
└─ 패킷을 br0로 보냄
└─ br0이 그 다음 처리 (라우팅)
```

### 5.2 Bridge와 Namespace

```
Host Namespace:
br0: 10.244.0.1/24
└─ IP 할당됨

Pod Namespace (Namespace A):
eth0: 10.244.0.5/24 (실제로는 veth-a)
└─ br0을 통해 연결된 상태
└─ br0에 할당된 IP를 gateway로 사용

라우팅 테이블 (Pod A 내):
$ ip route show
10.244.0.0/24 dev eth0 scope link src 10.244.0.5
default via 10.244.0.1 dev eth0  ← br0

의미:
├─ 10.244.0.0/24: 같은 네트워크 (br0을 통해)
└─ 그 외: 모두 br0으로 전송
```

---

## 📖 6. Linux Bridge의 포워딩 모드

### 6.1 Forwarding (포워딩)

```
Pod A → Pod B 통신:

FORWARDING MODE (기본):
Pod A ──→ br0 ──→ Pod B
         (포워딩)

패킷이 br0을 거쳐서 전달됨
특징:
└─ Pod들 사이의 통신만 br0을 거침
└─ 외부 네트워크는 별도 처리

$ ip link show br0
br0: <BROADCAST,MULTICAST,UP>
    └─ UP: 포워딩 활성화
```

### 6.2 Bridge와 ARP

```
Pod A가 Pod B의 MAC을 모를 때:

1️⃣ ARP 요청 (Broadcast):
   Pod A: "누가 10.244.0.6인가? MAC을 알려줘!"
   
   Frame:
   Src MAC: 02:42:ac:11:00:05 (Pod A)
   Dst MAC: FF:FF:FF:FF:FF:FF (Broadcast)
   ARP: Who has 10.244.0.6?

2️⃣ br0에서 Flooding:
   br0: "이건 broadcast다. 모든 포트로 전송"
   └─ veth-a는 이미 출발지니까 제외
   └─ veth-b, veth-c, eth0으로 전송

3️⃣ Pod B가 ARP 응답:
   Pod B: "내가 10.244.0.6이다. 내 MAC은 02:42:ac:11:00:06"
   
   Frame:
   Src MAC: 02:42:ac:11:00:06 (Pod B)
   Dst MAC: 02:42:ac:11:00:05 (Pod A)
   ARP: I have 10.244.0.6, MAC: ...

4️⃣ br0에서 학습 및 포워딩:
   학습: 02:42:ac:11:00:06 → veth-b
   포워딩: veth-a로 전송 (Pod A)

5️⃣ Pod A가 수신:
   ARP 캐시에 저장: 10.244.0.6 → 02:42:ac:11:00:06
```

---

## 📖 7. Linux Bridge와 veth의 통합

### 7.1 Bridge 포트로서의 veth

```
veth pair:
┌─────────────────────────────────────┐
│ veth-a ←→ veth-b                    │
│ (한쪽)    (다른 쪽)                 │
└─────────────────────────────────────┘

Bridge에 추가:
$ ip link set veth-a master br0

결과:
┌────────────────────────────────────┐
│ br0 (Bridge)                       │
│ ├─ veth-a (bridge port)           │
│ │   └─ peer: veth-b (Pod Namespace)│
│ ├─ veth-c (bridge port)           │
│ │   └─ peer: veth-d (Pod Namespace)│
│ └─ ...                            │
└────────────────────────────────────┘

특징:
├─ veth-a는 br0의 포트
├─ veth-b는 Pod의 eth0 (Bridge 모름)
├─ br0이 veth-a를 통해 veth-b와 통신
└─ Pod는 일반 네트워크 인터페이스 사용
```

### 7.2 Bridge와 veth의 성능

```
통신 경로별 오버헤드:

1️⃣ Pod A → Pod B (같은 Host):
   Pod A
   └─ eth0 (veth-b의 Host 측)
   └─ veth-a의 tx queue
   └─ br0 포워딩 (MAC 테이블 lookup)
   └─ veth-c의 rx queue
   └─ eth0 (veth-d의 Pod 측)
   └─ Pod B
   
   성능: ⭐⭐⭐⭐ (10-50μs)
   오버헤드: veth 2개 + bridge 로직

2️⃣ Pod A → 다른 Host의 Pod:
   Pod A
   └─ eth0 (veth-b)
   └─ br0 포워딩
   └─ eth0 (Host의 외부 NIC)
   └─ 네트워크 (오버헤드 커짐)
   └─ 다른 Host
   
   성능: ⭐⭐⭐ (100μs-10ms)
```

---

## 📖 8. Linux Bridge의 제어 플레인

### 8.1 Spanning Tree Protocol (STP)

```
Bridge 루프 방지 메커니즘:

루프 발생 상황:
       ┌─ br0 ─┐
       │        │
   veth-a    veth-b
       │        │
    ┌──┘        └──┐
    │              │
   Pod A ←→ Pod B  (아, eth 직접 연결?)

이런 상황에서 Broadcast가 무한 루프될 수 있음

STP 해결:
├─ Root Bridge 선택 (하나의 br0)
├─ Spanning Tree 구성 (루프 없는 구조)
└─ Blocked port: 일부 포트를 의도적으로 차단

Kubernetes에서:
└─ 일반적으로 STP 사용 안 함 (단순 토폴로지)
```

### 8.2 Bridge의 설정

```
br0 설정 확인:
$ brctl show
bridge name bridge id        STP enabled  interfaces
br0         8000.0242ac110001   no        veth-a
                                          veth-b
                                          veth-c

설정 변경:
$ brctl stp br0 on          # STP 활성화
$ brctl addif br0 veth-a    # 포트 추가 (ip link set도 가능)
$ brctl delif br0 veth-a    # 포트 제거
$ brctl showmacs br0        # MAC 테이블 확인
```

---

## 📖 9. Kubernetes와 Linux Bridge

### 9.1 cbr0 (Container Bridge 0)

```
각 Kubernetes Node의 기본 구성:

Node (Host OS):
┌────────────────────────────────────┐
│ eth0: 10.240.0.5 (Node IP)        │
│ cbr0: 10.244.0.1/24 (Bridge)      │
│ ├─ veth_pod_a                    │
│ │  └─ Pod A: 10.244.0.5           │
│ ├─ veth_pod_b                    │
│ │  └─ Pod B: 10.244.0.6           │
│ └─ veth_pod_c                    │
│    └─ Pod C: 10.244.0.7           │
│                                  │
│ Route:                           │
│ 10.244.0.0/24 dev cbr0          │ (로컬 Pod들)
│ 10.244.1.0/24 via 10.240.0.6    │ (다른 Node)
│ 10.244.2.0/24 via 10.240.0.7    │ (또 다른 Node)
│ ...                             │
└────────────────────────────────────┘
```

### 9.2 cbr0의 역할

```
역할 1️⃣: Pod 간 L2 연결
└─ Pod A ←─ veth ─→ Pod B (L2 switching)
└─ MAC 기반 포워딩

역할 2️⃣: 로컬 라우팅
└─ 10.244.0.0/24 out cbr0
└─ 로컬 네트워크로의 모든 트래픽을 cbr0으로

역할 3️⃣: 외부 통신의 게이트웨이
└─ Pod가 다른 네트워크로 나갈 때
└─ cbr0 (IP: 10.244.0.1)이 게이트웨이
└─ cbr0에서 라우팅 결정

역할 4️⃣: 모니터링
└─ Host에서 cbr0을 통한 트래픽 모니터링
└─ tcpdump -i cbr0
```

---

## 📖 10. Linux Bridge와 eBPF

### 10.1 eBPF를 통한 Bridge 모니터링

```
cbr0을 통과하는 모든 패킷 모니터링:

$ bpftrace -e 'kprobe:br_handle_frame {
  printf("Bridge 프레임: src=%x dst=%x\n", 
    arg1, arg2);
}'

Cilium의 경우:
└─ eBPF XDP로 cbr0의 패킷을 가로챔
└─ 성능 영향 없이 완벽한 가시성 제공
└─ NetworkPolicy 구현
```

### 10.2 eBPF를 통한 Bridge 최적화

```
기본 Bridge:
패킷 → br0 → MAC lookup → 포워딩 → 대상

eBPF 최적화 (Cilium):
패킷 → eBPF XDP 프로그램
        ├─ 정책 확인
        ├─ Drop (필요 없으면)
        ├─ Pass (일반 처리)
        └─ Redirect (직접 대상 포트로)

성능 개선:
└─ 불필요한 패킷을 XDP 단계에서 DROP
└─ br0의 MAC lookup 오버헤드 줄임
```

---

## 📖 11. Linux Bridge의 제한사항

### 11.1 성능 한계

```
Bridge 성능 병목:
1️⃣ MAC 테이블 Lookup
   └─ O(1)이지만, 해시 충돌 가능
   └─ 수백 개 MAC에서도 빠름

2️⃣ 모든 패킷이 br0을 거침
   └─ CPU 사용량 증가 가능
   └─ 대역폭 제한 가능

3️⃣ 단일 Host 제한
   └─ Bridge는 Host 내에서만 작동
   └─ 다른 Host의 Pod으로는 라우팅 필요
```

### 11.2 확장성 문제

```
문제: Pod 수가 많아질 때
└─ MAC 테이블 커짐
└─ Flooding 빈도 증가 (unknown destination)

해결책:
├─ 사전에 MAC 학습 (프로토콜 의존)
├─ Broadcast 최소화
├─ Multiple Bridge (복잡도↑)
└─ Overlay Network (Flannel 등, 차 장에서)
```

---

## 📖 12. Linux Bridge vs 다른 기술

### 12.1 Bridge vs veth pair 직접 연결

```
veth pair 직접:
Pod A ←─ veth pair ─→ Pod B
장점: 빠름 (br0 overhead 없음)
단점: N개 Pod = N*(N-1)/2 개 연결 필요 (O(N²))

Bridge 사용:
모든 Pod → br0
장점: N개 Pod = N개 연결만 필요 (O(N))
단점: br0의 overhead (미미함)

결론: Bridge가 훨씬 효율적
```

### 12.2 Bridge vs 라우터

```
Bridge (L2):
├─ MAC 기반 포워딩
├─ IP를 보지 않음
└─ 같은 네트워크 (서브넷) 내에서만

Router (L3):
├─ IP 기반 포워딩
├─ 라우팅 테이블 사용
└─ 서로 다른 네트워크 간 연결

Kubernetes:
└─ br0 (Bridge): 로컬 Pod들 (같은 Host)
└─ Linux Router: 다른 Host의 Pod들 (다음 장)
```

---

## 📖 13. 요약: Linux Bridge의 역할

### 한 문장 정의
**Linux Bridge는 여러 veth 인터페이스를 L2 레벨에서 연결하여 마치 물리 스위치처럼 작동하는 가상 장치다.**

### 핵심 역할

```
1️⃣ 연결 (L2 Switching)
   └─ MAC 주소 기반 포워딩
   └─ veth들을 같은 네트워크으로 통합

2️⃣ 확장성 (Scalability)
   └─ O(N) 확장성으로 많은 Pod 지원
   └─ veth direct 연결의 O(N²) 문제 해결

3️⃣ 투명성 (Transparency)
   └─ 애플리케이션은 일반 네트워크로 봄
   └─ Bridge의 존재는 숨겨짐

4️⃣ 게이트웨이 (Gateway)
   └─ br0의 IP가 기본 게이트웨이
   └─ 로컬 네트워크 내에서의 출구
```

### 다음 장으로의 연결

```
이제 이해한 것:
✅ Network Namespace로 격리
✅ veth pair로 연결
✅ Linux Bridge로 여러 개 통합

다음 장 (Routing):
→ "그럼 서로 다른 네트워크 (다른 Host)로는?"
→ "br0 밖의 통신은?"
```

---

## 🎓 핵심 개념 체크

- [ ] Linux Bridge가 무엇이고, 스위치와의 차이는?
- [ ] MAC 주소 학습 메커니즘은 어떻게 작동하는가?
- [ ] br0에 veth를 추가하면 어떻게 되는가?
- [ ] Flooding은 언제, 어떻게 발생하는가?
- [ ] ARP가 Bridge와 함께 어떻게 작동하는가?
- [ ] br0의 IP (10.244.0.1)의 의미는?
- [ ] cbr0과 일반 Linux Bridge의 차이는?
- [ ] Bridge의 성능은 어느 정도인가?

---

**다음 주제: Routing - "서로 다른 네트워크 간의 통신"** 🛣️
