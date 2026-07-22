# 13: Service Packet Flow (심층 패킷 추적)

## 🎯 핵심 질문

- **"Service IP로의 패킷이 정말 Pod에 도달하는가?"**
- NAT가 어떻게 투명하게 작동하는가?
- 응답 패킷은 어떻게 원래 Client로 돌아오는가?
- Service 로드밸런싱의 성능 문제는?

---

## 🔴 1. 왜? (Problem)

### Stateless LB 문제

```
❌ 단순 Round-Robin이면:

Client → Service → Pod A
Client → Service → Pod B
Client → Service → Pod C

문제 1: TCP 연결이 끊김
├─ SrcIP/Port 기반 상태 유지 필요
├─ RR은 같은 연결을 다른 Pod로 보낼 수 있음
└─ 결과: TCP 재설정/혼란

문제 2: 세션 데이터 손실
├─ Pod A에서 로그인한 사용자
├─ Pod B로 연결되면: 세션 없음
└─ 사용자 재로그인 필요

문제 3: 성능 저하
├─ 매번 Pod 변경 → TCP 핸드셰이크
├─ 레이턴시 증가
└─ 대역폭 낭비
```

### 해결: Connection Tracking

```
✅ Conntrack 사용:

Client → Service (처음)
  │ conntrack: 10.244.0.5 - Service → Pod A로 기록
  ↓
Client → Service (2번째)
  │ conntrack 조회: "이 연결은 Pod A로 가야 함"
  ↓
Client → Pod A로 자동 라우팅 (항상)

결과:
✅ 같은 연결 = 항상 같은 Pod
✅ 세션 유지
✅ 응답도 자동으로 돌아옴
```

---

## 🟡 2. 무엇? (Concept)

### Packet Flow의 각 단계

```
흐름 개요:

Client Pod (10.244.0.5)
  ↓ "Service IP로 요청 보낼게"
Network Stack
  ↓ (iptables PREROUTING)
DNAT: Service IP → Pod IP 변환
  ↓ (커널 라우팅 테이블)
Routing: "다른 호스트의 Pod IP네, 보내자"
  ↓ (CNI: cbr0 → veth → eth0)
Node의 네트워크 인터페이스
  ↓ (물리 네트워크)
Target Node
  ↓ (수신)
Target Pod
  ↓ (응답 생성)
POSTROUTING NAT (자동 역변환)
  ↓
Client Pod가 수신
```

### Connection Tracking (Conntrack)

```
Conntrack의 역할:

패킷 1: Client (10.244.0.5:12345) → Service (10.104.50.1:80)
  │ conntrack 생성:
  │ - Original: Client → Service
  │ - Reply: Pod A → Client
  │ - Timeout: 300초 (또는 설정값)
  ↓

NAT 변환:
  요청: 10.104.50.1:80 → 10.244.0.10:8080 (Pod A)
  응답: 10.244.0.10:8080 → 10.104.50.1:80 (자동 역변환)

다음 패킷 (같은 연결):
  conntrack 조회: "이 연결은 Pod A로 가야 함"
  → 자동으로 Pod A로 라우팅
```

---

## 🟢 3. 어떻게? (How)

### iptables 규칙 상세

```
1️⃣ PREROUTING 체인 (수신 패킷)

iptables -t nat -A PREROUTING \
  -p tcp -d 10.104.50.1 --dport 80 \
  -j DNAT --to-destination 10.244.0.10:8080

작동:
Packet: SrcIP=10.244.0.5, DstIP=10.104.50.1, DstPort=80
  │
  ├─ DNAT 규칙 매칭
  │
  └─ 변환: DstIP=10.244.0.10, DstPort=8080

2️⃣ OUTPUT 체인 (로컬 패킷)

Node의 애플리케이션이 Service로 접근할 때도 처리

3️⃣ POSTROUTING 체인 (송신 패킷)

iptables -t nat -A POSTROUTING \
  -j MASQUERADE  # 또는 SNAT

작동:
Pod → Client 응답 시 자동 역변환
SrcIP: 10.244.0.10:8080 → 10.104.50.1:80
(Conntrack이 자동으로 추적)
```

### 실제 패킷 추적 (tcpdump)

```
 Client 관점:
 $ tcpdump -i eth0 -n port 80
 10.244.0.5.12345 > 10.104.50.1.80: SYN
 10.104.50.1.80 > 10.244.0.5.12345: SYN-ACK
 10.244.0.5.12345 > 10.104.50.1.80: ACK
 (마치 Service에서 온 것처럼)

 Pod A 관점 (10.244.0.10):
 $ tcpdump -i eth0 -n port 8080
 10.244.0.5.12345 > 10.244.0.10.8080: SYN
 10.244.0.10.8080 > 10.244.0.5.12345: SYN-ACK
 (실제 주소가 보임)

 Node 관점 (iptables/conntrack):
 $ conntrack -L | grep 10.104.50.1
 tcp 6 300 ESTABLISHED \
  src=10.244.0.5 dst=10.104.50.1 \
  sport=12345 dport=80 \
  src=10.244.0.10 dst=10.244.0.5 \
  sport=8080 dport=12345 [ASSURED]
```

### 성능 특성

```
Conntrack 성능:

수명 (TTL): 기본 5분
상태 변화: NEW → ESTABLISHED → TIME_WAIT → CLOSED

Conntrack 테이블 크기 영향:
- 기본: 65536 항목 (시스템에 따라 다름)
- 초과 시: 패킷 드롭

Connection 맺기:
Short-lived (HTTP): 영향 적음
Long-lived (WebSocket): Conntrack 항목 장기 점유

Load Balancer로서의 성능:
- iptables: O(n) 성능 (규칙 개수에 선형)
- IPVS: O(1) 성능 (Conntrack 사용 동일)
```

---

## 🔵 4. 언제? (When)

### Service 패킷 흐름 최적화

```
상황 1: Short-lived 연결 (REST API)
→ iptables 충분
→ Conntrack 부하 낮음
→ 기본 설정 사용 가능

상황 2: Long-lived 연결 (WebSocket, gRPC)
→ Session Affinity 고려
→ Conntrack 항목 증가
→ timeoutSeconds 조정 필요

상황 3: 초당 만 개 이상 연결
→ Conntrack 메모리 증가
→ net.nf_conntrack_max 증가 필요
→ IPVS 모드 검토

상황 4: Pod 재시작 빈번
→ Conntrack 리셋됨
→ 기존 연결 끊김
→ 클라이언트 재연결 필요
```

### Conntrack 튜닝

```
Parameter: net.nf_conntrack_max
기본값: 65536
증가: net.netfilter.nf_conntrack_max=262144

Parameter: net.nf_conntrack_tcp_timeout_established
기본값: 432000 (5일)
감소: net.netfilter.nf_conntrack_tcp_timeout_established=600

Parameter: net.nf_conntrack_tcp_timeout_time_wait
기본값: 120초
용도: TIME_WAIT 상태 관리
```

---

## 💜 5. 실전 (Practice)

### Packet Flow 추적

```bash
# 1. Service 확인
kubectl get svc
# NAME    CLUSTER-IP      PORT
# nginx   10.104.50.1     80:30800/TCP

# 2. Pod 확인
kubectl get pods -o wide
# nginx-1   10.244.0.10
# nginx-2   10.244.0.11

# 3. iptables 규칙 확인
kubectl debug node/node1 -it --image=ubuntu
iptables -t nat -L -n | grep 10.104.50.1

# 4. Conntrack 확인
conntrack -L | grep 10.104.50.1 | head -5

# 5. tcpdump로 실제 추적
tcpdump -i eth0 -n 'port 80 or port 8080'
```

### 로드밸런싱 검증

```bash
# Session Affinity 없을 때 (RR)
for i in {1..10}; do
  kubectl exec pod1 -- curl -s 10.104.50.1:80
done
# Pod A, Pod B, Pod C, Pod A, ... 번갈아나타남

# Session Affinity 있을 때
kubectl patch svc nginx -p '{"spec":{"sessionAffinity":"ClientIP"}}'
for i in {1..10}; do
  kubectl exec pod1 -- curl -s 10.104.50.1:80
done
# Pod A, Pod A, Pod A ... 같은 Pod만
```

### Conntrack 모니터링

```bash
# Conntrack 통계
cat /proc/net/stat/nf_conntrack

# 활성 연결 수
wc -l /proc/net/nf_conntrack

# Service 관련 연결만
conntrack -L | grep 10.104 | wc -l

# 연결 상태별 분류
conntrack -L | grep -c ESTABLISHED
conntrack -L | grep -c TIME_WAIT
```

---

**다음**: NetworkPolicy (트래픽 제어) 🔒
