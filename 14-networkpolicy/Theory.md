# 14: NetworkPolicy (마이크로세그멘테이션)

## 🎯 핵심 질문

- **"왜 Pod 간에 모든 통신이 허용될까?"**
- NetworkPolicy가 없으면 무엇이 위험한가?
- NetworkPolicy는 어떻게 작동하는가?
- Kubernetes 내 보안은 어디서 시작되는가?

---

## 🔴 1. 왜? (Problem)

### 기본값: 모든 Pod이 서로 통신 가능

```
❌ NetworkPolicy 없을 때:

Namespace: production
├─ Pod: frontend (웹앱)
├─ Pod: backend (API)
├─ Pod: database (DB)
└─ Pod: malicious (악의적 Pod)

현재 상태:
모든 Pod이 모든 Pod과 통신 가능!
  malicious → database:5432
  malicious → backend:3000
  malicious → frontend:80
  (모두 차단 불가)

위험:
1. 데이터 유출
   └─ malicious가 DB에 접근 → 고객 정보 탈취

2. 서비스 중단
   └─ malicious가 backend 포트 스캔 → DoS

3. 감염 확산
   └─ 한 Pod 침해 → 전체 서비스 침해
```

### NetworkPolicy로 해결

```
✅ NetworkPolicy 적용:

Namespace: production (모든 수신 금지)
├─ frontend: client에서만 80 수신 허용
├─ backend: frontend에서만 3000 수신 허용
├─ database: backend에서만 5432 수신 허용
└─ malicious: 아무도 접근 불가

결과:
malicious → database: DROP
malicious → backend: DROP
frontend → backend: ACCEPT (정책 허용)
backend → database: ACCEPT (정책 허용)
```

---

## 🟡 2. 무엇? (Concept)

### NetworkPolicy의 구성 요소

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  # 1️⃣ 정책 적용 대상
  podSelector:
    matchLabels:
      app: backend
  
  # 2️⃣ 정책 타입
  policyTypes:
  - Ingress      # 수신 제어
  - Egress       # 송신 제어
  
  # 3️⃣ 수신 규칙
  ingress:
  - from:        # 누구에게서 오는 트래픽 허용?
    - podSelector:
        matchLabels:
          app: frontend
    ports:       # 어느 포트를 허용?
    - protocol: TCP
      port: 3000
  
  # 4️⃣ 송신 규칙
  egress:
  - to:          # 누구에게 보내는 트래픽 허용?
    - podSelector:
        matchLabels:
          app: database
    ports:       # 어느 포트를 허용?
    - protocol: TCP
      port: 5432
```

### 정책 평가 로직

```
Pod1 → Pod2로 패킷 전송 시:

1️⃣ Pod2의 Ingress 정책 확인
   ├─ Pod2에 NetworkPolicy 있음?
   ├─ "Ingress" policyType 있음?
   └─ from 규칙이 Pod1을 허용?
   
2️⃣ Pod1의 Egress 정책 확인
   ├─ Pod1에 NetworkPolicy 있음?
   ├─ "Egress" policyType 있음?
   └─ to 규칙이 Pod2를 허용?
   
3️⃣ 모두 허용이면: ACCEPT
   하나라도 거부면: DROP
```

### 기본값 (Default Policy)

```
NetworkPolicy 없음:
모든 Ingress/Egress: ALLOW
(전통적 Kubernetes)

Deny-All Ingress 정책:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}           # 모든 Pod에 적용
  policyTypes:
  - Ingress
  ingress: []               # 규칙 없음 = 아무것도 허용 안 함
(이 정책 이후 명시적 allow 필요)

Deny-All Egress 정책:
결과: Pod가 외부로 통신 불가 (정책에 없으면)
```

---

## 🟢 3. 어떻게? (How)

### CNI 플러그인별 구현

```
🟡 Calico (기본)

구현: iptables + ipset
  1. NetworkPolicy → iptables 규칙 변환
  2. Pod IP들을 ipset에 저장 (효율성)
  3. FORWARD 체인에 규칙 추가

성능:
  - 100 정책: ~1ms 평가 시간
  - 1000 정책: ~5ms (성능 저하)
  - 설정 변경: 수 초 (규칙 재생성)

🔵 Cilium (eBPF 기반)

구현: eBPF 프로그램 + 커널 훅
  1. NetworkPolicy → eBPF 바이트코드 컴파일
  2. NIC 인터페이스에 직접 attach
  3. 커널 레벨에서 즉시 평가

성능:
  - 모든 규칙: ~0.1ms (일정)
  - 10000 정책도 동일 성능
  - 설정 변경: 수십 ms (eBPF 리컴파일)

✅ Weave

구현: 자체 vxlan + 필터링
  성능: 중간 수준
  복잡도: 높음
```

### 실제 iptables 규칙 (Calico)

```bash
# Calico가 생성하는 규칙 예:

iptables -A cali-wl-out-<pod-id> \
  -p tcp --dport 3000 \
  -m set --match-set cali40:allowedFrontend src \
  -j ACCEPT

iptables -A cali-wl-in-<pod-id> \
  -j DROP  # Default Deny

# 결과:
# Pod가 다른 Pod에서의 트래픽을 받으려면
# 반드시 allowedFrontend ipset에 있어야 함
```

### 정책 작동 흐름 (Cilium)

```
1️⃣ NetworkPolicy 생성
$ kubectl apply -f backend-policy.yaml
  ↓
2️⃣ API Server → etcd 저장
  ↓
3️⃣ Cilium Agent 감지
  ↓
4️⃣ eBPF 프로그램 생성
  ├─ Pod IP 조회
  ├─ 허용 규칙 매칭 로직
  └─ eBPF 바이트코드로 컴파일
  ↓
5️⃣ 각 Node에 배포
  ├─ Pod의 veth에 attach
  ├─ Ingress/Egress 훅 설치
  └─ 실시간 트래픽 필터링
  ↓
6️⃣ 패킷 도착
  ├─ veth → eBPF 프로그램 실행
  ├─ 규칙 매칭 (nanoseconds)
  ├─ ACCEPT 또는 DROP
  └─ 커널 스택으로 진행
```

---

## 🔵 4. 언제? (When)

### NetworkPolicy 적용 시나리오

```
상황 1: 개발 환경
→ NetworkPolicy 불필요
→ 모든 Pod이 테스트 목적으로 통신 필요
→ "Allow All" 사용

상황 2: 프로덕션 (다중 팀)
→ "Deny All Ingress" 기본
→ 명시적으로 필요한 것만 Allow
→ 팀별 네임스페이스 격리

상황 3: 보안 요구사항 강화
→ Egress도 제어 (데이터 유출 방지)
→ L7 정책 (Cilium만 가능)
→ API 기반 정책 (특정 엔드포인트만 허용)

상황 4: 멀티테넌트 클러스터
→ 각 테넌트별 "Deny All"
→ 테넌트 간 통신 차단
→ 중앙 서비스만 cross-tenant 허용
```

### 정책 패턴

```
패턴 1: "Deny All" 기본 (Zero Trust)

ApiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  ingress: []
  egress: []  # 아무것도 허용 안 함

→ 모든 Pod이 고립됨
→ 필요한 것만 명시적으로 허용

패턴 2: "Tier-based" (계층 기반)

frontend: 
  - 클라이언트 트래픽만 수신 (80, 443)
  - backend로만 송신 (3000)

middleware:
  - frontend에서만 수신 (3000)
  - database로만 송신 (5432)
  - external API 호출 (443)

database:
  - middleware에서만 수신 (5432)
  - 송신 없음
```

---

## 💜 5. 실전 (Practice)

### NetworkPolicy 생성 및 검증

```bash
# 1. 모든 Pod이 통신 가능 확인
kubectl run client --image=nginx
kubectl run target --image=nginx
kubectl exec client -- curl -s target:80
# 성공

# 2. Deny All Ingress 적용
cat > deny-all.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
EOF
kubectl apply -f deny-all.yaml

# 3. 이제 통신 실패
kubectl exec client -- curl -s target:80
# 타임아웃 (DROP됨)

# 4. 선택적 허용 추가
cat > allow-client.yaml << EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-client
spec:
  podSelector:
    matchLabels:
      run: target
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: client
    ports:
    - protocol: TCP
      port: 80
EOF
kubectl apply -f allow-client.yaml

# 5. 이제 다시 성공
kubectl exec client -- curl -s target:80
# 성공 (policy에 명시됨)
```

### 정책 디버깅

```bash
# 1. NetworkPolicy 확인
kubectl get networkpolicy -A
kubectl describe networkpolicy deny-all

# 2. Pod 레이블 확인
kubectl get pods --show-labels

# 3. 정책이 적용되었는지 확인 (Calico)
kubectl debug node/node1 -it --image=ubuntu
iptables -L -t filter | grep cali
grep -r "backend" /var/lib/calico/  # Calico 데이터

# 4. 정책이 적용되었는지 확인 (Cilium)
kubectl exec -n kube-system cilium-xxxxx -- cilium policy list
kubectl exec -n kube-system cilium-xxxxx -- cilium endpoint list

# 5. 패킷 추적
kubectl exec client -- tcpdump -i eth0 -n
# SYN만 보이고 SYN-ACK 없으면: 정책으로 DROP됨
```

### 정책 성능 측정

```bash
# 규칙 없을 때 처리량
kubectl run server --image=nginx
kubectl run load --image=nicolaka/netshoot
kubectl exec load -- iperf3 -c server -t 10
# 예: 1000 Mbps

# Deny All 적용
kubectl apply -f deny-all.yaml
# (서비스 중단)

# 선택적 Allow
kubectl apply -f allow-policy.yaml
kubectl exec load -- iperf3 -c server -t 10
# 예: 995 Mbps (거의 손실 없음)

# Cilium과 Calico 비교
# Cilium eBPF: 0.5% 오버헤드
# Calico iptables: 2-5% 오버헤드 (정책 개수에 따라)
```

---

**다음**: eBPF 고급 기술 🚀
