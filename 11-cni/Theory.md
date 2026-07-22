# 11: CNI (Container Network Interface)

## 🎯 핵심 질문

- **"kubelet이 어떻게 Pod에 IP를 할당할까?"**
- CNI가 왜 별도의 플러그인인가?
- CNI 플러그인 간 성능/기능 차이는?
- CNI 선택이 성능에 미치는 영향?

---

## 🔴 1. 왜? (Problem)

### 네트워킹을 kubelet이 직접 하면?

```
❌ 나쁜 설계:

kubelet이 모든 네트워킹 담당하면:
├─ Flannel을 지원하려면 kubelet 수정
├─ Calico를 지원하려면 또 수정
├─ Cilium을 지원하려면 또 수정
│  (매번 kubelet 코드 변경 + 테스트)
│
└─ 결과: kubelet이 거대하고 복잡해짐
   └─ 유지보수 어려움
   └─ 네트워킹 버그가 코어 영향
   └─ 새로운 플러그인 추가 어려움
```

### CNI로 분리하면?

```
✅ 좋은 설계:

CNI = 표준 인터페이스

kubelet                    Network Plugin
  │                               │
  │ "Pod 생성했어"                │
  │ (네트워킹 설정 필요)          │
  ├─────────────────────────────→ │
  │ (JSON 형식 요청)               │
  │                               │
  │ ← (성공: IP할당, 라우팅설정)  │
  │

장점:
✅ kubelet은 "네트워킹 설정하기" 요청만 함
✅ 네트워킹 구현은 플러그인 책임
✅ 플러그인 교체 = 단순히 파일 교체
✅ 새로운 플러그인 개발 독립적
```

---

## 🟡 2. 무엇? (Concept)

### CNI 정의

```
CNI (Container Network Interface)

= 컨테이너의 네트워킹을 위한 표준 인터페이스

구성 요소:
1️⃣ 스펙 (JSON)
   └─ 네트워크 설정 명세

2️⃣ 실행 파일 (Executable)
   └─ /opt/cni/bin/flannel
   └─ /opt/cni/bin/calico
   └─ /opt/cni/bin/cilium

3️⃣ 플러그인 (Plugin)
   └─ 위 실행 파일 + 백엔드 데몬
   └─ Flannel (flanneld)
   └─ Calico (calico-node)
```

### CNI 플러그인 역할

```
CNI 플러그인이 해야 할 일:

1️⃣ IP 할당 (IPAM: IP Address Management)
   └─ 어느 IP를 사용할지 결정
   └─ 충돌 없도록 관리
   └─ 예: 10.244.0.5 할당

2️⃣ Interface 생성
   └─ veth 생성
   └─ 내부: Pod Namespace
   └─ 외부: Host Namespace
   └─ 예: Pod eth0 ↔ Host veth1234

3️⃣ Bridge 연결
   └─ veth를 bridge에 연결
   └─ 예: cbr0에 veth1234 추가

4️⃣ 라우팅 설정
   └─ Pod에서 외부로 나가는 경로 설정
   └─ 다른 Node로 가는 경로 설정
   └─ 예: 10.244.1.0/24 → via 172.16.0.6

5️⃣ DNS 설정
   └─ Pod의 /etc/resolv.conf 작성
   └─ 예: nameserver 10.96.0.10 (coredns)
```

---

## 🟢 3. 어떻게? (How)

### CNI 호출 순서

```
kubectl run mypod --image=nginx
        ↓
API Server → etcd에 Pod 저장
        ↓
kubelet이 감지: "새 Pod 생성해야 함"
        ↓
containerd 호출 (CRI)
        ↓
containerd: Network Namespace 생성
        ↓
kubelet: "/opt/cni/bin/flannel" 실행
        ↓
        JSON 입력:
        {
          "cniVersion": "0.4.0",
          "name": "cbr0",
          "type": "flannel",
          "pod": "mypod",
          "namespace": "default",
          ...
        }
        ↓
flannel 플러그인:
  1. IP 할당: 10.244.0.5
  2. veth 생성
  3. cbr0에 연결
  4. 라우팅 설정
        ↓
        JSON 출력:
        {
          "cniVersion": "0.4.0",
          "interfaces": [...],
          "ips": [{"address": "10.244.0.5/24"}],
          "routes": [...]
        }
        ↓
kubelet: 결과 확인, Pod 시작 계속
        ↓
Pod 실행 시작
```

### 실제 파일 구조

```
/etc/cni/net.d/10-flannel.conflist
───────────────────────────────────
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}

/opt/cni/bin/flannel
────────────────────
실행 파일 (flannel 플러그인)
호출 형식: flannel ADD/DEL/CHECK
```

### 성능 특성

```
CNI 플러그인별 성능:

Flannel (VXLAN)
├─ 오버헤드: 높음 (UDP 캡슐화)
├─ CPU: 중간~높음
├─ 레이턴시: 1-5ms (다른 호스트)
└─ 트루풋: 제한적 (캡슐화 때문)

Calico (BGP)
├─ 오버헤드: 낮음 (직접 라우팅)
├─ CPU: 낮음~중간
├─ 레이턴시: 0.5-2ms (다른 호스트)
└─ 트루풋: 높음 (네이티브 라우팅)

Cilium (eBPF)
├─ 오버헤드: 매우 낮음
├─ CPU: 낮음 (커널 레벨)
├─ 레이턴시: 0.1-0.5ms (다른 호스트)
└─ 트루풋: 매우 높음 (최적화)

실제 측정 (Pod 간 ping RTT):
같은 호스트:     Flannel ≈ Calico ≈ Cilium (모두 ~0.1ms)
다른 호스트:     Flannel ~2ms > Calico ~1ms > Cilium ~0.5ms
```

---

## 🔵 4. 언제? (When)

### CNI 선택 기준

```
선택 기준 1: 성능 요구도

낮음 (개발/테스트)
  → Flannel
  → "간단함"이 최고 장점
  → Pod 간 통신 충분

중간 (프로덕션 일반)
  → Calico
  → 성능 + 정책 + 안정성
  → 가장 널리 사용

높음 (고성능 요구)
  → Cilium
  → 최고 성능 + L7 정책
  → 마이크로서비스 최적화

─────────────────────────

선택 기준 2: 기능 요구도

Pod 통신만:        Flannel
+ NetworkPolicy:   Calico / Cilium
+ L7 정책:         Cilium만 가능
+ 클라우드 통합:   Azure CNI

─────────────────────────

선택 기준 3: 클라우드

AWS EKS           Calico / AWS VPC CNI
Azure AKS         Azure CNI / Calico
GCP GKE           Calico (기본)
온프레미스        Flannel / Calico / Cilium
```

### 플러그인 비교표

```
기능              Flannel  Calico  Cilium  Azure CNI
──────────────────────────────────────────────────
설정 복잡도        ⭐⭐    ⭐⭐    ⭐⭐⭐  ⭐
Pod 통신          ✅      ✅      ✅      ✅
NetworkPolicy      ❌      ✅      ✅      ✅
L7 정책           ❌      ❌      ✅      ❌
성능 (레이턴시)    ⭐⭐    ⭐⭐⭐  ⭐⭐⭐⭐ ⭐⭐⭐⭐
BGP               ❌      ✅      ❌      ❌
eBPF              ❌      ❌      ✅      ❌
클라우드 연동      ❌      일부    일부    ✅
커뮤니티 지원      좋음    매우좋음 좋음   Microsoft
```

---

## 💜 5. 실전 (Practice)

### CNI 설치 및 검증

```bash
# 현재 CNI 확인
kubectl get daemonset -n kube-system
kubectl get pods -n kube-system | grep -i flannel

# Flannel 설치
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Calico 설치
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/master/manifests/tigera-operator.yaml

# CNI 구성 확인
ls /etc/cni/net.d/
cat /etc/cni/net.d/10-flannel.conflist

# CNI 바이너리 확인
ls /opt/cni/bin/
```

### Pod에 IP가 할당되지 않는 경우 진단

```bash
# 1단계: CNI 플러그인 상태 확인
kubectl get daemonset -n kube-system
kubectl logs -n kube-system -l app=flannel | tail -20

# 2단계: kubelet 로그 확인
kubectl debug node/node1 -it --image=ubuntu
journalctl -u kubelet | grep -i cni

# 3단계: Pod 상태 확인
kubectl describe pod mypod
# "pending" 상태면 CNI가 IP를 할당하지 못함

# 4단계: 네트워크 구성 확인
ip addr show cbr0
ip route show
```

### CNI 성능 벤치마크

```bash
# Pod 간 통신 레이턴시 측정
kubectl exec -it pod1 -- ping -c 100 10.244.1.5 | grep avg

# 대역폭 측정 (iperf3)
kubectl exec -it pod1 -- iperf3 -c <pod2-ip> -t 10

# 패킷 손실률
kubectl exec -it pod1 -- ping -c 1000 10.244.1.5 | grep packet
```

---

**다음**: kube-proxy와 Service 구현 🎯
