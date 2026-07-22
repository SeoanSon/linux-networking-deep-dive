# 10: Kubernetes Networking (Pod IP 모델)

## 🎯 핵심 질문

- **"왜 모든 Pod에 고유 IP를 줘야 하는가?"**
- 각 Pod이 IP를 받지 않으면 무엇이 문제인가?
- Kubernetes의 Pod IP 모델은 어떻게 작동하는가?
- Docker와 Kubernetes의 네트워킹 차이는?

---

## 🔴 1. 왜? (Problem)

### 기존 Docker 방식의 한계

```
Docker Swarm / 일반 Container Orchestration:

Container 1 (192.168.1.10)
Container 2 (192.168.1.11)
Container 3 (192.168.1.12)
    ↓ (다 같은 호스트에 있으면...)

문제 1: 포트 충돌
- Container 1: :8080
- Container 2: :8080 (충돌!)
- 반드시 다른 포트 할당 필요

문제 2: 서비스 디스커버리 복잡
- Container가 옮겨지면 IP/Port 변경
- 클라이언트는 새 주소를 계속 추적해야 함

문제 3: 다중 호스트 확장성
- 호스트 간 통신 시 NAT/Port Mapping 복잡
- 성능 오버헤드

문제 4: 네트워크 정책 적용 어려움
- IP 기반 정책을 세우기 어려움
- 포트 기반으로만 제어 가능
```

### Kubernetes가 필요한 이유

```
🎯 목표: "모든 Pod이 자신의 고유 IP를 가지고,
         마치 같은 네트워크에 있는 것처럼 직접 통신"

이렇게 되면:
✅ 포트 충돌 없음 (각 Pod: 독립적 포트 공간)
✅ 서비스 디스커버리 단순 (IP는 불변)
✅ 다중 호스트 확장성 (NAT 없는 직접 통신)
✅ 네트워크 정책 강력 (IP 기반 제어)
✅ 마이크로서비스 친화적 (서비스 간 직접 통신)
```

### 비용 (Trade-off)

```
대신 감수해야 할 것:
❌ CNI 구성의 복잡성 (Flannel, Calico, Cilium 중 선택)
❌ IP 주소 관리 (Pod CIDR 범위 계획)
❌ 오버레이 네트워크의 성능 오버헤드 (Flannel의 경우)
❌ 네트워크 문제 디버깅 복잡성
```

---

## 🟡 2. 무엇? (Concept)

### Kubernetes 네트워킹 모델

```
Kubernetes는 다음을 보장 (RFC):

1️⃣ Pod마다 고유 IP
   └─ Pod A: 10.244.0.5
   └─ Pod B: 10.244.1.7
   └─ 클러스터 내 모든 Pod이 유일

2️⃣ Pod 간 직접 통신 (NAT 없음)
   Pod A (10.244.0.5) → Pod B (10.244.1.7)
   └─ 직접 통신, IP 변환 없음

3️⃣ Node와 Pod 직접 통신 (NAT 없음)
   Node 1 (172.16.0.5) → Pod (10.244.0.5)
   └─ 직접 통신, 별도 Port Mapping 없음

4️⃣ Pod 내부 프로세스는 호스트 포트를 사용하지 않음
   Pod 내 app: :8080 (Pod IP에만 바인드)
   └─ 호스트의 8080 포트와 무관
```

### 용어 정의

```
Pod IP (Cluster IP)
└─ 각 Pod에 할당되는 IP
└─ 범위: 10.244.0.0/16 (기본)
└─ 동적 할당 (CNI 플러그인이 담당)
└─ Pod 재시작 시 변경 가능 (따라서 Service 필요)

Service IP (Virtual IP)
└─ 안정적인 엔드포인트
└─ Pod 그룹을 대표하는 IP
└─ 다음 장에서 상세히

Node Network
└─ 노드 자신의 IP (172.16.0.5 등)
└─ 호스트 운영체제가 할당
└─ 안정적, 재시작해도 유지
```

---

## 🟢 3. 어떻게? (How)

### 내부 동작 메커니즘

```
Pod 생성 시 네트워킹 구성 순서:

1️⃣ kubelet이 Pod 요청 수신
   kubectl run mypod --image=nginx
        ↓
   API Server → etcd 저장
        ↓
   kubelet이 Node에서 감지

2️⃣ Container Runtime 호출 (CRI)
   kubelet → containerd/docker
        ↓
   "Pod의 Namespace 생성해줘"
        ↓
   Network Namespace 생성

3️⃣ CNI 플러그인 호출
   kubelet → CNI executable
        ↓
   "Pod에 IP 할당하고 네트워크 구성해줘"

4️⃣ CNI가 구체적으로 수행
   ├─ Pod에 veth 생성
   ├─ Host의 cbr0에 연결
   ├─ IP 할당 (IPAM)
   ├─ 라우팅 설정
   └─ /etc/resolv.conf 설정

5️⃣ Application 시작
   kubelet → containerd → App process
        ↓
   Pod IP로 접근 가능
```

### 실제 Linux 구조

```
Host 관점:

eth0 (Node NIC)
  └─ 172.16.0.5

cbr0 (Bridge)
  ├─ veth1234@if2 (Pod A 연결)
  ├─ veth5678@if2 (Pod B 연결)
  └─ 10.244.0.1/24

Pod A 관점:
  └─ eth0@if10 (10.244.0.5)
  └─ lo (127.0.0.1)
  └─ 라우팅: 0.0.0.0/0 → 10.244.0.1 (cbr0)

Pod B 관점:
  └─ eth0@if20 (10.244.1.5)
  └─ lo (127.0.0.1)
  └─ 라우팅: 0.0.0.0/0 → 10.244.1.1 (cbr0)
```

### 성능 특성

```
Pod 간 통신 레이턴시 (Flannel UDP):
약 0.5-2ms (같은 호스트)
약 1-5ms (다른 호스트, 오버레이)

Pod 간 통신 레이턴시 (Calico BGP):
약 0.1-0.5ms (같은 호스트)
약 0.5-2ms (다른 호스트, 직접 라우팅)

병목 지점:
1. CNI 플러그인 선택 (Flannel < Calico < Native)
2. 오버레이 vs 라우트 기반
3. MTU 설정 (기본 1500)
```

---

## 🔵 4. 언제? (When)

### 사용 시점 & 상황별 선택

```
상황: 개발/테스트
→ Flannel (간단, 충분)
→ Pod 간 통신만 필요
→ 성능 비중 낮음

상황: 프로덕션 (일반 워크로드)
→ Calico (안정성 + 성능 + NetworkPolicy)
→ Pod 간 통신 + 정책 제어 필요
→ 멀티테넌트 격리 필요

상황: 프로덕션 (고성능 요구)
→ Cilium (최고 성능 + L7 정책)
→ 마이크로서비스 + API 게이트웨이
→ eBPF 이용한 고급 기능

상황: 클라우드 네이티브 (AKS)
→ Azure CNI (VNet 직접 사용)
→ Pod이 Azure VNet IP 사용
→ 클라우드 정책 연동 필요
```

### 대안 비교

```
                Flannel    Calico    Cilium    Azure CNI
설정 난이도      ⭐        ⭐⭐      ⭐⭐⭐    ⭐
성능            ⭐⭐      ⭐⭐⭐    ⭐⭐⭐⭐  ⭐⭐⭐⭐
NetworkPolicy   ❌       ✅        ✅        ✅
L7 정책        ❌       ❌        ✅        ❌
오버레이        ✅       BGP       eBPF      라우팅
클라우드 연동    ❌       일부      일부      ✅
```

---

## 💜 5. 실전 (Practice)

### AKS에서의 Pod IP 모델

```
AKS는 두 가지 네트워킹 옵션 제공:

옵션 1: Kubenet (기본)
- Pod CIDR: 10.244.0.0/16 (내부)
- Node CIDR: 10.240.0.0/12 (VNet)
- Pod은 NAT 뒤에 (외부 직접 접근 불가)
- 간단하지만 성능 낮음

옵션 2: Azure CNI (권장)
- Pod IP = Azure VNet IP (10.0.0.0/8)
- Pod이 VNet에 직접 참여
- 클라우드 정책 연동 가능
- 성능 우수

클러스터 생성 시:
az aks create \
  --network-plugin azure \        # Azure CNI 사용
  --network-policy calico \       # NetworkPolicy 지원
  --service-cidr 10.104.0.0/12 \  # Service IP 범위
  --docker-bridge-address 172.17.0.1/16
```

### Pod 네트워킹 검증

```bash
# Pod 생성
kubectl run test --image=nginx

# Pod IP 확인
kubectl get pods -o wide
# NAME   READY  IP          NODE
# test   1/1    10.244.0.5  node1

# Pod 내부에서 확인
kubectl exec test -- ip addr
# eth0: 10.244.0.5/24
# lo: 127.0.0.1

# Host에서 확인
kubectl debug node/node1 -it --image=ubuntu
# ip addr show cbr0
# ip route show
```

### 문제 진단

```bash
# 1. Pod이 IP를 받지 못함
kubectl describe pod test | grep -i ip

# 2. CNI 플러그인 상태
kubectl get daemonset -n kube-system

# 3. Node의 라우팅 확인
kubectl debug node/node1 -it --image=ubuntu
# ip route show | grep 10.244

# 4. Pod 간 통신 테스트
kubectl exec pod1 -- ping -c 1 10.244.1.5
```

---

**다음**: CNI 플러그인 선택과 설정 🔌
