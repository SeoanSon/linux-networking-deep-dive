# 06: iptables (패킷 필터링) 이론

## 🎯 핵심

**iptables**는 Linux의 방화벽 엔진입니다. 패킷을 필터링하고, NAT를 하고, 규칙을 적용합니다.

```
iptables의 역할:

패킷 도착
    ↓
iptables 규칙 확인
    ├─ ACCEPT: 통과
    ├─ DROP: 버림 (응답 없음)
    ├─ REJECT: 거부 (응답 함)
    └─ REDIRECT: 다른 포트로
    ↓
패킷 처리 또는 삭제
```

## 📖 1. iptables의 구조

### Tables와 Chains

```
iptables 구조:
3개의 Tables (용도별)
  ├─ filter table (기본, 필터링)
  ├─ nat table (주소 변환)
  └─ mangle table (패킷 수정)

각 Table은 여러 Chains를 가짐:
filter:
  ├─ INPUT (들어오는 패킷)
  ├─ FORWARD (지나가는 패킷)
  └─ OUTPUT (나가는 패킷)

예: 
$ sudo iptables -L -t filter
$ sudo iptables -L -t nat
```

### Chain의 작동

```
INPUT chain (Host로 들어오는 패킷):

패킷 도착 → INPUT chain 진입
         ↓
         규칙 1: 포트 22? → ACCEPT
         규칙 2: 포트 80? → ACCEPT
         규칙 3: 기타? → DROP (기본정책)
         ↓
         처리 결정

OUT FORWARD chain (다른 호스트로 가는 패킷):

패킷 → FORWARD chain 진입
     ↓
     규칙 1: 10.244.0.0/24 → ACCEPT
     규칙 2: 8.8.8.8? → ACCEPT
     규칙 3: 기타? → DROP
     ↓
     처리 결정
```

## 📖 2. iptables 규칙

### 규칙 추가

```bash
# 포트 8080 접근 허용
sudo iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Pod 네트워크 간 통신 허용
sudo iptables -A FORWARD -s 10.244.0.0/24 -d 10.244.1.0/24 -j ACCEPT

# 기본정책 변경
sudo iptables -P INPUT DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -P FORWARD DROP
```

### 규칙 확인

```bash
$ sudo iptables -L -n

Chain INPUT (policy ACCEPT)
target     prot opt source               destination
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0      tcp dpt:22
ACCEPT     tcp  --  0.0.0.0/0            0.0.0.0/0      tcp dpt:80
DROP       all  --  0.0.0.0/0            0.0.0.0/0

Chain FORWARD (policy ACCEPT)
ACCEPT     all  --  10.244.0.0/24        10.244.1.0/24
DROP       all  --  0.0.0.0/0            0.0.0.0/0
```

## 📖 3. NAT (Network Address Translation)

### SNAT (Source NAT)

```
외부로 나가는 Pod의 출발지 IP 변환:

Pod (10.244.0.5) → 외부 인터넷 (8.8.8.8)

1️⃣ Pod에서 패킷 생성
   Src IP: 10.244.0.5 (Pod IP)
   Dst IP: 8.8.8.8

2️⃣ Node의 eth0에서 SNAT 규칙 적용
   Src IP: 10.240.0.5 (Node IP)로 변환
   Dst IP: 8.8.8.8 (변경 없음)

3️⃣ 인터넷으로 전송
   응답: 8.8.8.8 → 10.240.0.5

4️⃣ Node에서 응답 수신
   역 NAT: Src IP를 10.244.0.5로 변환

5️⃣ Pod가 수신
   Src IP: 8.8.8.8 (원본)
   Dst IP: 10.244.0.5 (Pod IP)

규칙:
$ sudo iptables -t nat -A POSTROUTING -s 10.244.0.0/24 ! -d 10.244.0.0/24 -j MASQUERADE
```

### DNAT (Destination NAT)

```
외부 → Service로의 목적지 IP 변환:

외부 (203.0.113.1:80) → Service (10.104.50.1:80)

1️⃣ 외부에서 패킷 생성
   Src IP: 203.0.113.1
   Dst IP: 10.104.50.1 (Service IP)

2️⃣ Node에서 DNAT 적용
   Src IP: 203.0.113.1 (변경 없음)
   Dst IP: 10.244.0.5 (Pod IP)로 변환

3️⃣ Pod로 전송

4️⃣ Pod에서 응답
   역 DNAT: Dst IP를 10.104.50.1로 변환

규칙:
$ sudo iptables -t nat -A PREROUTING -p tcp -d 10.104.50.1 --dport 80 -j DNAT --to-destination 10.244.0.5:8080
```

## 📖 4. Kubernetes와 iptables

### kube-proxy와 iptables

```
Kubernetes Service는 iptables로 구현됨 (기본):

Service 생성: 
kubectl create service clusterip myservice --tcp=80:8080

kube-proxy가 자동으로 iptables 규칙 생성:
$ sudo iptables -t nat -L -n | grep -i service

KUBE-SERVICES
KUBE-SVC-... (각 Service마다)
KUBE-SEP-... (각 Endpoint마다)

작동:
외부 또는 Pod → Service IP:Port
             ↓
             kube-proxy의 iptables 규칙
             ↓
             Pod IP:Port로 변환 및 라우팅
```

### iptables 규칙 예

```
Service: nginx, ClusterIP 10.104.50.1, Port 80
Pod: 10.244.0.5:8080, 10.244.0.6:8080

생성되는 규칙:
1. -A KUBE-SERVICES -d 10.104.50.1/32 -p tcp --dport 80 -j KUBE-SVC-...
2. -A KUBE-SVC-... -m statistic --mode random --probability 0.5 -j KUBE-SEP-pod1
3. -A KUBE-SVC-... -j KUBE-SEP-pod2
4. -A KUBE-SEP-pod1 -p tcp -j DNAT --to-destination 10.244.0.5:8080
5. -A KUBE-SEP-pod2 -p tcp -j DNAT --to-destination 10.244.0.6:8080

작동:
트래픽 → 10.104.50.1:80 (Service)
      ↓ (50% 확률)
      10.244.0.5:8080 (Pod 1) 또는 10.244.0.6:8080 (Pod 2)
```

## 📖 5. NetworkPolicy와 iptables

```
NetworkPolicy 예:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: block-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

이것을 처리하는 CNI 플러그인이 iptables 규칙 생성:
$ sudo iptables -L -n

(Calico 등이 자동으로 규칙 생성)
```

---

## 🎓 핵심

- iptables는 Linux 방화벽의 중심
- Tables (filter, nat, mangle) → Chains (INPUT, FORWARD, OUTPUT) 구조
- kube-proxy가 Service를 iptables 규칙으로 구현
- NetworkPolicy도 iptables (또는 eBPF) 규칙으로 구현

**다음: netfilter - iptables 뒤의 커널 엔진** 🔌
