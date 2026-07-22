# 07: netfilter (커널 구현) 이론

## 🎯 핵심

**netfilter**는 Linux 커널의 네트워크 패킷 처리 엔진입니다. iptables는 netfilter를 제어하는 사용자 공간 도구입니다.

```
사용자 관점: iptables (명령어)
           ↓
커널 관점: netfilter (코드)

관계: iptables ← (제어) → netfilter
```

## 📖 1. netfilter Hook Points

### 5가지 Hook

```
NF_IP_PRE_ROUTING    (0)  ← 패킷 도착, 라우팅 전
          ↓
      라우팅
          ↓
        ↙ ↘
NF_IP_FORWARD      NF_IP_INPUT     ← 로컬 프로세스로
  (1)  (2)
   ↓    ↓
   ↓   로컬 프로세스
   ↓    ↓
NF_IP_POST_ROUTING← (3)
  (4)
   ↓
외부로 나감
```

### Hook별 역할

```
1. PRE_ROUTING (도착 직후)
   └─ DNAT (목적지 IP 변환)
   └─ 라우팅 전 처리

2. INPUT (로컬 수신 전)
   └─ 로컬 프로세스로 가는 패킷 필터링
   └─ 방화벽

3. FORWARD (전달)
   └─ 다른 호스트로 가는 패킷
   └─ 라우팅 후

4. OUTPUT (로컬 송신)
   └─ 로컬 프로세스에서 나가는 패킷
   └─ 송신 방화벽

5. POST_ROUTING (송신 전)
   └─ SNAT (출발지 IP 변환)
   └─ 송신 직전
```

## 📖 2. netfilter 구현

### 커널 코드 위치

```
Linux Kernel:
net/netfilter/
├─ core.c              # Hook 관리
├─ nf_conntrack.c      # 연결 추적
├─ nf_nat.c            # NAT
├─ nf_queue.c          # 패킷 큐
└─ nf_sockopt.c        # socket 옵션
```

### Hook 등록

```c
// netfilter hook 구조
struct nf_hook_ops {
    nf_hookfn *hook;        // 실행할 함수
    struct module *owner;
    u_int8_t pf;            // 프로토콜 패밀리
    unsigned int hooknum;   // 0-4 (5개 hook)
    int priority;           // 우선순위
};

// Hook 함수 반환값
NF_DROP         = 0         // 패킷 버림
NF_ACCEPT       = 1         // 통과
NF_STOLEN       = 2         // 처리함
NF_QUEUE        = 3         // 사용자공간으로
NF_REPEAT       = 4         // 재실행
```

## 📖 3. 연결 추적 (Conntrack)

### stateful firewall

```
기본 방화벽 (stateless):
모든 패킷을 규칙에 따라 필터링

Conntrack (stateful):
양방향 통신 상태를 추적

예:
클라이언트 → 서버:80 (명시적 규칙 필요)
서버 → 클라이언트 (자동으로 허용, 기존 연결로 인식)

작동:
1️⃣ C→S 패킷 도착
   └─ 규칙 확인: "80 포트 ACCEPT"
   └─ Conntrack: 새 연결로 등록

2️⃣ S→C 응답 도착
   └─ Conntrack 확인: "이미 알려진 연결"
   └─ 규칙 재확인 없이 ACCEPT
   └─ 성능 향상

상태 종류:
NEW        : 새 연결
ESTABLISHED: 기존 연결
RELATED    : 관련된 연결 (FTP data 등)
INVALID    : 잘못된 상태
```

---

## 📖 4. 요약

netfilter는 Linux 커널의 패킷 처리 엔진:
- 5개의 Hook Point
- 각 Hook에서 패킷 필터링/수정
- Conntrack으로 연결 추적
- iptables와 nftables가 이를 제어

**다음: Containers - 컨테이너 네트워킹 통합** 📦
