# 03: veth pairs (가상 이더넷 페어) 이론

## 🎯 핵심 질문

- **"격리된 Network Namespace들이 어떻게 통신할까?"**
- veth pair는 무엇이고, 왜 필요할까?
- 실제로 "케이블"처럼 작동하나?
- Kubernetes Pod이 외부와 통신하는 물리적 메커니즘은?

---

## 📖 1. veth pair란?

### 정의
**veth (Virtual Ethernet)**는 **두 개의 격리된 Network Namespace를 연결하는 가상 케이블**입니다.

```
일반 이더넷 케이블:
┌────────────┐ 물리 케이블 ┌────────────┐
│ 컴퓨터 A   │───────────│ 컴퓨터 B   │
│ eth0:MAC-1 │  RJ45/Cat5│ eth0:MAC-2 │
└────────────┘           └────────────┘


veth pair (가상):
┌──────────────────┐  ┌──────────────────┐
│ Namespace 1      │  │ Namespace 2      │
│                  │  │                  │
│ veth0 ←─────────→ veth1               │
│ (한쪽 끝)│  │  │ (다른 쪽 끝)│
│                  │  │                  │
└──────────────────┘  └──────────────────┘
```

### 특징

```
✅ veth pair의 특징:

1️⃣ 항상 쌍으로 존재
   └─ veth0과 veth1 같은 형태
   └─ 한쪽을 삭제하면 다른쪽도 자동 삭제

2️⃣ 한쪽 끝은 Namespace 1에, 다른 쪽은 Namespace 2에
   └─ 각 Namespace가 자신의 끝만 봄
   └─ "케이블의 양 끝"

3️⃣ 패킷이 한쪽에서 들어가면 다른쪽에서 나옴
   └─ 데이터가 투과됨
   └─ 자동으로 라우팅 없이 연결

4️⃣ 성능: 매우 빠름
   └─ 커널 메모리 내에서만 작동
   └─ 물리 네트워크 거치지 않음

5️⃣ 독립적인 MAC 주소
   └─ veth0: MAC-A
   └─ veth1: MAC-B
   └─ (서로 다름)
```

---

## 📖 2. veth pair의 역사와 필요성

### 2.1 언제 등장했나?

```
2006년 ┌─ Network Namespace 개발 시작
       │  (Eric W. Biederman)

2007년 │  Network Namespace의 기본 기능만 있음
       │  문제: Namespace 내에서 lo만 사용 가능
       │  └─ 외부와 통신할 방법이 없음!
       ├─ veth 필요성 대두

2008년 │  veth device 개발 시작

2009년 │  Linux 2.6.33에서 veth 정식 포함
       │  "Virtual Ethernet interface"
       └─ Network Namespace과 함께 사용 가능
```

### 2.2 등장하기 전 문제

```
veth 없이는:
┌──────────────────┐  ┌──────────────────┐
│ Namespace A      │  │ Namespace B      │
│                  │  │                  │
│ IP: 10.0.0.1     │  │ IP: 10.0.0.2     │
│ (격리됨)        │  │ (격리됨)        │
│                  │  │                  │
│ lo (loopback만)  │  │ lo (loopback만)  │
│ 외부 연결 없음   │  │ 외부 연결 없음   │
└──────────────────┘  └──────────────────┘
        ❌                      ❌
      통신 불가              통신 불가

각 Namespace가 "고립된 섬"처럼 작동
```

### 2.3 veth의 해결책

```
veth pair 추가 후:
┌──────────────────────────────────────────┐
│ Host Namespace                           │
│ ┌────────────────┐        ┌────────────┐ │
│ │ Namespace A    │ veth-A → veth-B   └──→ Bridge/Router
│ │ 10.0.0.1       │        │          │
│ │ eth0: veth-A   │        │ Host에서 │
│ └────────────────┘        │ 관리 가능 │
│          ↑                 │          │
│          │ veth pair (케이블)  │          │
│          ↓                 │          │
│ ┌────────────────┐        │          │
│ │ Namespace B    │ veth-C → veth-D  └──→ Bridge
│ │ 10.0.0.2       │        │          │
│ │ eth0: veth-C   │        │          │
│ └────────────────┘        │          │
└──────────────────────────────────────────┘

✅ 이제 각 Namespace가 외부와 통신 가능!
```

---

## 📖 3. veth pair의 작동 원리

### 3.1 내부 구조 (커널 관점)

```
Linux 커널 메모리:

┌─────────────────────────────────────────┐
│ veth pair 데이터 구조                    │
│                                         │
│ struct net_device veth0                 │
│ {                                       │
│   name: "veth0"                         │
│   MAC: 02:42:ac:11:00:02               │
│   partner: ─────┐                      │
│   namespace: [NS A] │                  │
│   rx_queue                              │
│   tx_queue                              │
│   ...                                   │
│ }                                       │
│                                         │
│ struct net_device veth1        ←────┐  │
│ {                                   │  │
│   name: "veth1"                    │  │
│   MAC: 02:42:ac:11:00:03          │  │
│   partner: ──────────────────────┘  │
│   namespace: [NS B]                  │
│   rx_queue                           │
│   tx_queue                           │
│   ...                                │
│ }                                    │
│                                      │
│ 연결: veth0.tx_queue ← → veth1.rx_queue
│                                      │
└─────────────────────────────────────┘
```

### 3.2 패킷 흐름 (한 방향)

```
Namespace A에서 Namespace B로 패킷 전송:

1️⃣ 애플리케이션이 데이터 전송
   Application A ──→ socket ──→ 10.0.0.2에 패킷 전송 요청

2️⃣ Namespace A의 네트워킹 스택
   Namespace A ──→ 라우팅 테이블 확인
   └─ "10.0.0.2는 veth0을 통해 갈 수 있다"

3️⃣ veth0으로 패킷 전송
   Namespace A: eth0 (실제로는 veth0)
   Namespace A의 veth0 ──→ 데이터 queue에 저장

4️⃣ 커널이 자동으로 veth1로 전달
   veth0.tx_queue ──→ veth1.rx_queue (직접 복사)

5️⃣ Namespace B의 veth1에서 데이터 도착
   Namespace B: eth0 (실제로는 veth1)
   └─ Namespace B의 관점: "eth0에서 패킷이 도착했다"

6️⃣ Namespace B의 네트워킹 스택 처리
   Namespace B ──→ 라우팅 테이블 확인
   └─ "이건 내 IP (10.0.0.2)다. 애플리케이션에 전달"

7️⃣ 애플리케이션이 데이터 수신
   Application B ←─ socket ←─ Namespace B
```

### 3.3 양방향 동작

```
실제로는 양방향으로 작동:

Namespace A                          Namespace B
Application A                        Application B
    ↑ ↓                                ↑ ↓
Socket  ←───→ veth0 ←──────────→ veth1 ←───→ Socket
                 ↑                ↓
                 │ (tx_queue)     (rx_queue)
                 │                │
                 └────────┬────────┘
                  veth pair
                  (양방향 전송)

Namespace A → Namespace B:
veth0.tx → veth1.rx

Namespace B → Namespace A:
veth1.tx → veth0.rx

🔄 독립적으로 양방향 통신 가능
```

---

## 📖 4. veth pair의 성능 특성

### 4.1 성능 비교

```
통신 방식별 성능 (지연시간, 처리량):

1️⃣ 같은 Namespace 내 loopback:
   localhost:8000 ←→ localhost:9000
   성능: ⭐⭐⭐⭐⭐ 극도로 빠름 (< 1μs)
   설명: 같은 메모리 공간, context switching 없음

2️⃣ veth pair (같은 Host):
   Namespace A ←─veth─→ Namespace B
   성능: ⭐⭐⭐⭐ 매우 빠름 (1-10μs)
   설명: 커널 메모리 내, 한두 번의 복사

3️⃣ Bridge를 통한 통신:
   Pod A ←─veth─→ cbr0 ←─veth─→ Pod B
   성능: ⭐⭐⭐⭐ 빠름 (10-50μs)
   설명: bridge 로직 추가

4️⃣ 다른 Host의 통신:
   Pod A (Node 1) → 네트워크 → Pod B (Node 2)
   성능: ⭐⭐⭐ 보통 (100μs - 10ms)
   설명: 물리 네트워크 타고 감, 지연시간 증가

5️⃣ 인터넷 통신:
   Pod (AKS) → Azure 클라우드 → 외부 인터넷
   성능: ⭐⭐ (1ms - 1s)
   설명: 인터넷 레이턴시
```

### 4.2 대역폭 (Throughput)

```
veth pair의 대역폭은 Host 네트워크에 의존:

Host 물리 네트워크: 10 Gbps
    ↓
veth pair: 10 Gbps (물리 네트워크와 동일)
    └─ 한계: Host의 네트워크 카드 속도

예시:
물리 NIC: 1 Gbps
└─ veth 최대: 1 Gbps

물리 NIC: 10 Gbps
└─ veth 최대: 10 Gbps

물리 NIC: 100 Gbps (고성능)
└─ veth 최대: 100 Gbps

✅ veth overhead는 거의 없음
```

### 4.3 CPU 사용량

```
veth pair 패킷 처리:

1️⃣ 패킷이 veth0 rx_queue에 도착
   └─ CPU: ~50-100 사이클

2️⃣ 커널이 veth1 tx_queue로 복사
   └─ CPU: ~100-200 사이클 (패킷 크기에 따라)

3️⃣ veth1에서 인터럽트 처리
   └─ CPU: ~50-100 사이클

합계: ~200-400 사이클 / 패킷
(최신 CPU는 GHz 단위이므로 매우 빠름)

비교:
물리 NIC: ~1000+ 사이클 (하드웨어 처리)
veth pair: ~200-400 사이클 (완전 소프트웨어)

✅ veth는 물리 NIC보다 효율적!
```

---

## 📖 5. Linux 커널에서의 veth 구현

### 5.1 커널 소스 위치

```
Linux Kernel Source Tree:
drivers/net/veth.c     (메인 구현 파일)

주요 데이터 구조:
struct veth_priv {
    struct net_device *peer;    # 상대방 veth
    atomic64_t dropped;         # 드롭된 패킷 수
    struct bpf_prog *bpf_xdp;   # eBPF XDP 프로그램
    ...
};

struct veth_q_stat {
    u64 packets;
    u64 bytes;
    struct u64_stats_sync syncp;
};
```

### 5.2 veth 생성 과정 (커널 관점)

```
$ ip link add veth0 type veth peer name veth1

1️⃣ rtnl_link_ops.newlink() 호출
   └─ veth_newlink() 함수 실행

2️⃣ 두 개의 net_device 구조체 할당
   ├─ veth0용 메모리
   └─ veth1용 메모리

3️⃣ 서로를 가리키도록 설정
   veth0.peer = veth1
   veth1.peer = veth0

4️⃣ 각각의 tx/rx 큐 초기화
   veth0: {tx_queue, rx_queue}
   veth1: {tx_queue, rx_queue}

5️⃣ MAC 주소 생성
   veth0: random MAC-A (02:42:ac:11:00:02)
   veth1: random MAC-B (02:42:ac:11:00:03)

6️⃣ 네트워크 인터페이스 등록
   ├─ veth0를 Host에 등록
   └─ veth1을 Namespace B에 이동

7️⃣ 완료
   $ ip link show
   veth0: ←→ peer: veth1
   veth1: ←→ peer: veth0
```

### 5.3 패킷 처리 코드 흐름

```
veth0에서 패킷 송신 시:

veth_xmit() 함수:
  ├─ 패킷 받음
  ├─ peer device 확인
  │  └─ peer = veth1
  ├─ veth1의 rx_queue에 넣기
  ├─ veth1의 프로세스를 깨우기
  └─ 완료 (매우 빠름)

반대 방향도 동일:
veth1에서 송신 → veth0의 rx_queue
```

---

## 📖 6. veth pair와 Namespace

### 6.1 veth와 Network Namespace 연결

```
생성:
$ sudo ip netns add ns1
$ sudo ip netns add ns2
$ sudo ip link add veth-a type veth peer name veth-b

초기 상태:
veth-a: Host Namespace에 있음
veth-b: Host Namespace에 있음

분리:
$ sudo ip link set veth-b netns ns2

결과:
Host Namespace: veth-a
Namespace 1 (ns1): (뭔가 없음)
Namespace 2 (ns2): veth-b

IP 할당:
$ sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth-b
$ sudo ip netns exec ns2 ip link set veth-b up

Namespace 2에서 조회:
$ sudo ip netns exec ns2 ip link show
1: lo: <LOOPBACK>
2: veth-b: <BROADCAST,MULTICAST,UP>
   └─ 10.0.0.2
```

### 6.2 Kubernetes Pod과 veth

```
Pod 생성 과정에서 veth 사용:

1️⃣ Container runtime (containerd)
   ├─ 새 Network Namespace 생성
   └─ Process 시작

2️⃣ CNI 플러그인 (예: Flannel, Calico)
   ├─ veth pair 생성
   │  └─ 한쪽은 Pod Namespace로, 다른쪽은 Host로
   ├─ IP 주소 할당
   │  └─ 10.244.x.x 형태
   └─ Bridge에 연결

3️⃣ 결과:
   Host Namespace:
   ├─ eth0: 172.16.0.5 (Node IP, 외부 통신용)
   ├─ cbr0: 10.244.0.0/24 (Container Bridge)
   └─ veth12345: 10.244.0.5 (Pod A로)
   └─ veth67890: 10.244.0.6 (Pod B로)

   Pod Namespace:
   ├─ eth0: 10.244.0.5 (실제로는 veth12345)
   └─ lo: 127.0.0.1

4️⃣ 통신:
   Pod A (eth0 = veth12345)
   ←─ veth12345 ─→ cbr0 ←─ veth67890 ─→ Pod B (eth0)
```

---

## 📖 7. veth와 MAC 주소

### 7.1 MAC 주소의 역할

```
veth pair 생성:
$ ip link add veth0 type veth peer name veth1

결과:
$ ip link show | grep veth
3: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN>
   link/ether 02:42:ac:11:00:02 brd ff:ff:ff:ff:ff:ff
4: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN>
   link/ether 02:42:ac:11:00:03 brd ff:ff:ff:ff:ff:ff

MAC 주소의 의미:
02:42:ac:11:00:02
├─ 02: 로컬 관리 비트 (Locally Administered)
├─ 42: Docker/veth 특정 식별자
├─ ac:11:00: 네트워크 부분 (172.17.0.0)
└─ 02: 인터페이스 부분

동일 Namespace 내에서:
└─ MAC 주소는 상대방을 식별하는 용도 (ARP)

다른 Namespace 사이:
└─ MAC은 veth에 의해 투명하게 처리됨
   (각 Namespace는 자신의 MAC만 관심)
```

### 7.2 Bridge와 MAC 주소

```
Container Bridge (cbr0) 구조:

Namespace A              Host                Namespace B
┌─────────────┐      ┌────────────┐      ┌─────────────┐
│   Pod A     │      │ Host OS    │      │   Pod B     │
│ eth0        │      │ cbr0       │      │ eth0        │
│ MAC:02:42..02│─veth─┤ (bridge)   │─veth─│ MAC:02:42..03│
│ IP:10.244.0.5│      │ MAC:02:42..01│      │ IP:10.244.0.6│
└─────────────┘      └────────────┘      └─────────────┘

Bridge의 역할:
1️⃣ veth0과 veth1을 연결
2️⃣ MAC 주소를 학습 (MAC table)
3️⃣ 프레임을 올바른 포트로 포워딩

MAC table:
02:42:ac:11:00:02 → veth0 (Pod A)
02:42:ac:11:00:03 → veth1 (Pod B)
02:42:ac:11:00:01 → cbr0 (Host)

✅ Bridge는 MAC 주소 기반으로 스위칭
```

---

## 📖 8. veth pair의 제한사항

### 8.1 격리와 연결의 트레이드오프

```
veth pair의 특성:

✅ 격리:
└─ 각 Namespace가 독립적인 네트워크 스택

✅ 연결:
└─ 명시적으로만 통신 가능 (veth를 통해)

❌ 자동 격리 불가능:
└─ veth로 연결되면 완전 통신 가능
└─ 세밀한 제어는 iptables/NetworkPolicy 필요

❌ 성능:
└─ 패킷마다 커널 메모리 복사 오버헤드
└─ 매우 작지만, 고성능이 필요하면 문제 가능
```

### 8.2 디버깅의 어려움

```
문제 상황:

Pod A ←─veth─→ Pod B 통신이 안 됨

확인해야 할 것들:
1️⃣ veth가 actually 연결되었나?
   $ ip link show | grep veth

2️⃣ IP 주소가 올바르게 할당되었나?
   $ ip addr show

3️⃣ Namespace가 올바른가?
   $ ip netns exec [pod] ip addr

4️⃣ Bridge가 패킷을 전달하나?
   $ tcpdump -i cbr0

5️⃣ iptables 규칙이 막았나?
   $ iptables -L -n

이 중 하나를 실수하면 통신 실패
```

---

## 📖 9. veth와 다른 네트워킹 기술의 비교

### 9.1 veth vs 다른 격리 방식

```
1️⃣ veth pair (Namespace 간 연결)
   사용: Pod ←→ Pod, Host ←→ Pod
   속도: ⭐⭐⭐⭐ (매우 빠름)
   용도: 로컬 연결

2️⃣ VLAN (가상 LAN)
   사용: 물리 네트워크 분리
   속도: ⭐⭐⭐ (물리 속도)
   용도: 엔터프라이즈 네트워킹

3️⃣ VPN (가상 사설망)
   사용: 암호화된 인터넷 연결
   속도: ⭐⭐ (암호화 오버헤드)
   용도: 보안 통신

4️⃣ Overlay Network (Flannel 등)
   사용: 서로 다른 Host의 Pod 연결
   속도: ⭐⭐⭐ (캡슐화 오버헤드)
   용도: 클러스터 간 연결

5️⃣ Host Network Mode
   사용: Pod가 Host Namespace 사용
   속도: ⭐⭐⭐⭐⭐ (가장 빠름)
   용도: 고성능 필요시 (권장 안 함)
```

### 9.2 veth vs TAP/TUN

```
veth pair:
├─ 커널 네트워킹 스택을 사용
├─ 일반적인 네트워크 인터페이스처럼 작동
├─ 성능: 빠름
└─ 권장: Kubernetes에서 standard

TAP (네트워크 인터페이스):
├─ 사용자 공간과 커널을 연결
├─ VPN, QEMU VM에서 사용
├─ 성능: 느림 (context switching)
└─ 권장: 특수한 경우만

TUN (라우팅 인터페이스):
├─ IP 레벨에서 작동
├─ VPN에서 많이 사용
├─ 성능: 중간
└─ 권장: 특수한 경우만
```

---

## 📖 10. veth와 eBPF

### 10.1 eBPF를 통한 veth 모니터링

```
eBPF 프로그램으로 veth 패킷 추적:

$ bpftrace -e 'kretprobe:veth_xmit { 
  printf("패킷 전송: %d bytes\n", retval); 
}'

결과:
패킷 전송: 64 bytes
패킷 전송: 1500 bytes
패킷 전송: 100 bytes
...

Cilium의 경우:
└─ veth를 통한 모든 패킷을 eBPF로 모니터링
└─ 성능에 영향 없이 완벽한 가시성 제공
```

### 10.2 eBPF를 통한 veth 최적화

```
일반적인 veth:
패킷 → veth0 → 커널 처리 → veth1 → 애플리케이션

eBPF XDP 최적화:
패킷 → XDP 프로그램 ← 직접 결정
        ├─ Drop (필요 없으면)
        ├─ Pass (일반 처리)
        └─ Redirect (다른 인터페이스로)

성능 개선:
❌ 일반: 모든 패킷 처리
✅ eBPF: 필요한 패킷만 처리

예: 10000 packets/sec
일반: 10000 packets 처리
eBPF: 필요한 500 packets만 처리 (95% DROP)
```

---

## 📖 11. AKS에서의 veth 구조

### 11.1 실제 AKS Pod의 veth

```
AKS Node 구조:

Host Namespace (Node):
├─ eth0: 10.240.0.4 (Node IP, Azure VNet)
├─ cbr0: 10.244.0.1 (Container Bridge)
└─ veth pairs: (각 Pod마다)
   ├─ veth_00000a: 10.244.0.5 (Pod A로)
   ├─ veth_00000b: 10.244.0.6 (Pod B로)
   ├─ veth_00000c: 10.244.0.7 (Pod C로)
   └─ veth_0000...: ...

Pod A Namespace:
├─ eth0: 10.244.0.5 (실제로는 veth_00000a)
├─ lo: 127.0.0.1
└─ 라우팅: 10.244.0.0/24 via 10.244.0.1 (cbr0)

Pod B Namespace:
├─ eth0: 10.244.0.6 (실제로는 veth_00000b)
├─ lo: 127.0.0.1
└─ 라우팅: 10.244.0.0/24 via 10.244.0.1 (cbr0)
```

### 11.2 AKS Pod 간 통신 경로 (veth 기준)

```
Pod A에서 Pod B로 통신:

1️⃣ Pod A의 애플리케이션 (IP 10.244.0.6 로 전송)
2️⃣ Pod A의 Namespace 네트워킹 스택
3️⃣ eth0 (= veth_00000a의 Namespace 측)
4️⃣ veth_00000a로 패킷 전송
5️⃣ ─────── veth pair 연결 ─────────
6️⃣ cbr0에서 veth_00000a의 Host 측 도착
7️⃣ cbr0에서 MAC 테이블 조회
   └─ 10.244.0.6은 veth_00000b임을 학습
8️⃣ 패킷을 veth_00000b로 포워딩
9️⃣ veth_00000b의 Namespace 측으로 전달
🔟 Pod B의 Namespace 네트워킹 스택
🔟+1 eth0 (= veth_00000b의 Namespace 측) 에서 수신
🔟+2 Pod B의 애플리케이션이 패킷 수신

총 지연시간: ~10-50μs (평균)
```

---

## 📖 12. veth 디버깅 명령어

### 12.1 veth 상태 확인

```
Host에서:
$ ip link show
3: veth0@veth1: <BROADCAST,MULTICAST,UP>
   link/ether 02:42:ac:11:00:02

$ ip addr show veth0
inet 10.244.0.5/24 scope global veth0

$ ethtool -S veth0
NIC statistics:
     peer_ifindex: 4  ← 상대방 인터페이스 번호
     ...

Container에서:
$ ip link show
2: eth0: <BROADCAST,MULTICAST,UP>
   link/ether 02:42:ac:11:00:02 (Host와 같은 MAC)

$ ip route show
10.244.0.0/24 dev eth0 scope link src 10.244.0.5
default via 10.244.0.1 dev eth0
```

### 12.2 veth 트래픽 모니터링

```
# 실시간 패킷 확인
$ tcpdump -i veth0 -n

# 특정 프로토콜 필터
$ tcpdump -i veth0 icmp

# 상세한 분석
$ tcpdump -i veth0 -nn -vvv

# 출력을 파일로 저장
$ tcpdump -i veth0 -w capture.pcap
$ wireshark capture.pcap
```

---

## 📖 13. 요약: veth pair의 역할

### 한 문장 정의
**veth pair는 격리된 Network Namespace들 사이에 가상의 양방향 연결을 제공하는 소프트웨어 기반 인터페이스다.**

### 세 가지 핵심 역할

```
1️⃣ 연결 (Connection)
   └─ 격리된 Namespace들을 서로 통신하게 함
   └─ "가상 케이블"역할

2️⃣ 격리 유지 (Isolation Preservation)
   └─ 연결하면서도 각 Namespace의 격리 유지
   └─ 명시적 연결만 가능

3️⃣ 투명성 (Transparency)
   └─ 애플리케이션은 일반 네트워크 인터페이스처럼 봄
   └─ 그 뒤의 가상화는 숨겨짐
```

### 다음 장으로의 연결

```
이제 이해한 것:
✅ Network Namespace가 격리되는 방식
✅ veth pair로 두 Namespace가 연결되는 방식

다음 장 (Linux Bridge):
→ "여러 개의 veth들을 어떻게 연결하지?"
→ "많은 Pod들이 모두 통신하려면?"
```

---

## 🎓 핵심 개념 체크

이 장을 이해했으면 다음에 답할 수 있어야 합니다:

- [ ] veth pair가 무엇이고, 왜 가상 "케이블"인가?
- [ ] veth의 두 끝이 어디에 위치하는가?
- [ ] 패킷이 veth를 통해 어떻게 이동하는가?
- [ ] veth0에서 송신한 패킷이 veth1에서 수신되는 메커니즘은?
- [ ] veth pair의 성능은 어느 정도인가?
- [ ] Linux 커널에서 veth는 어떻게 구현되는가?
- [ ] AKS Pod에서 veth의 역할은?
- [ ] Pod A에서 Pod B로의 패킷 경로에서 veth의 위치는?
- [ ] veth와 MAC 주소의 관계는?
- [ ] Bridge가 veth와 함께 작동하는 방식은?

---

## 📚 더 알아보기

### 관련 Linux 명령어 (다음 Hands-on에서)

```bash
# veth pair 생성
sudo ip link add veth0 type veth peer name veth1

# Namespace로 이동
sudo ip link set veth1 netns ns2

# IP 주소 할당
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth1
sudo ip netns exec ns2 ip link set veth1 up

# 통신 테스트
sudo ip netns exec ns2 ping 10.0.0.1

# veth 삭제
sudo ip link del veth0
```

### 커널 소스 참고

```
Linux Kernel: drivers/net/veth.c
주요 함수:
- veth_newlink()      # veth pair 생성
- veth_xmit()         # 패킷 송신
- veth_receive()      # 패킷 수신
- veth_change_mtu()   # MTU 변경
```

---

**다음 주제: Linux Bridge - 여러 veth를 연결하는 L2 스위칭** 🌉
