# 12: kube-proxy (Service 구현)

## 🎯 핵심 질문

- **"Service IP는 실제로 어디에 존재하는가?"**
- kube-proxy가 왜 필요한가?
- iptables vs IPVS의 성능 차이는?
- Service로 들어온 트래픽이 어떻게 Pod에 도달하는가?

---

## 🔴 1. 왜? (Problem)

### Pod IP 모델만으로는 부족한 이유

```
❌ Pod IP만 사용하면:

문제 1: Pod IP는 동적
├─ Pod 재시작 → 새 IP 할당
├─ Auto-scaling → Pod 증가/감소
├─ 클라이언트는 항상 최신 Pod IP 추적 필요
└─ 매우 불편

문제 2: 많은 Pod에 균등하게 분산 필요
├─ 3개 Pod이 있으면?
├─ 클라이언트가 3개 IP를 모두 알아야?
├─ 각각에 요청 분산하는 코드 필요?
└─ 복잡함

문제 3: Pod 내부 서비스를 외부에 노출?
├─ 클러스터 외부 클라이언트?
├─ Pod IP는 프라이빗
├─ 어떻게 외부에 서비스할 것?

📊 Docker 조회 예시:
$ docker ps
container1 (192.168.1.10:8080)
container2 (192.168.1.11:8080)
container3 (192.168.1.12:8080)

클라이언트: "어느 걸 써야 하지?"
```

### Service로 해결

```
✅ Service = 안정적 엔드포인트

Service 생성:
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: 10.104.50.1      # 안정적 VirtualIP
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nginx                  # label로 Pod 선택

결과:
✅ 클라이언트: 10.104.50.1:80으로 고정 접근
✅ Pod 추가/제거 → Service는 불변
✅ Pod 재시작 → Service는 불변
✅ 자동 로드밸런싱
✅ 내부/외부 모두 지원
```

---

## 🟡 2. 무엇? (Concept)

### Service 타입

```
1️⃣ ClusterIP (기본)
   └─ 클러스터 내부만 접근
   └─ Service IP: 10.104.x.x (내부)
   └─ 예: Pod ↔ Pod 통신

2️⃣ NodePort
   └─ 각 Node의 포트로 노출
   └─ 외부 클라이언트 접근 가능
   └─ 예: Client → 172.16.0.5:30080 → Service → Pod

3️⃣ LoadBalancer
   └─ 클라우드 로드밸런서 통합
   └─ 외부 IP 할당
   └─ 예: Client → 52.178.5.1:80 → Service → Pod

4️⃣ ExternalName
   └─ 외부 서비스 이름 매핑만
   └─ 실제 라우팅 없음
```

### kube-proxy 역할

```
Service 생성 시 흐름:

API Server
    ↓ (Service 이벤트)
kube-proxy (각 Node에서 실행)
    ↓
1. Service 감지
2. 해당 Pod들 (Endpoints) 조회
3. 규칙 생성 (iptables 또는 IPVS)
4. Service IP:Port → Pod IP:Port 매핑
    ↓
결과: Service IP로의 요청이 자동으로 Pod에 전달됨
```

### Service IP의 특성

```
Service IP (Cluster IP)
├─ 실제 네트워크 인터페이스에 바인드되지 않음
│  (단순히 iptables 규칙일 뿐)
│
├─ 예: 10.104.50.1
│  이 IP는 어떤 Node의 eth0에도 없음
│  하지만 "마치 있는 것처럼" 동작
│
├─ iptables에 의해 자동 변환됨
│  10.104.50.1:80 → 10.244.0.5:8080 (Pod A)
│  10.104.50.1:80 → 10.244.0.6:8080 (Pod B)
│
└─ 각 Node의 kube-proxy가 규칙을 생성
   따라서 클러스터의 모든 위치에서 접근 가능
```

---

## 🟢 3. 어떻게? (How)

### kube-proxy 모드별 구현

```
┌─ iptables 모드 (기본)
│
│ 작동:
│ 1. Service 생성 감지
│ 2. iptables 규칙 추가
│    $ iptables -A KUBE-SERVICES -p tcp -d 10.104.50.1 --dport 80 \
│        -j KUBE-SVC-XXXXX
│    $ iptables -A KUBE-SVC-XXXXX -p tcp -j DNAT \
│        --to-destination 10.244.0.5:8080
│ 3. 커널의 NAT가 자동 처리
│
│ 장점: 간단
│ 단점: 규칙이 많으면 느림 (O(n) 성능)
│       500+ Service면 문제 시작
│
├─ IPVS 모드 (고성능)
│
│ 작동:
│ 1. Service 생성 감지
│ 2. Virtual Service 생성
│    $ ipvsadm -A -t 10.104.50.1:80 -s rr
│ 3. Real Server 등록
│    $ ipvsadm -a -t 10.104.50.1:80 -r 10.244.0.5:8080 -m
│    $ ipvsadm -a -t 10.104.50.1:80 -r 10.244.0.6:8080 -m
│
│ 장점: O(1) 성능, 5000+ Service 지원
│       다양한 로드밸런싱 알고리즘
│ 단점: 약간 복잡
│
└─ userspace 모드 (레거시)
  작동: kube-proxy 프로세스가 직접 패킷 처리
  단점: 매우 느림, 추천하지 않음
```

### 실제 트래픽 흐름

```
Client Pod (10.244.0.10)
    ↓ "10.104.50.1:80으로 요청"
  [패킷]
    ↓ (Node 커널의 netfilter)
  iptables/IPVS 규칙 매칭
    ↓ "10.244.0.5:8080으로 변환"
  [패킷] 원본IP: 10.244.0.10, 목적지: 10.244.0.5:8080
    ↓ (라우팅)
  Node의 cbr0 / 다른 Node로
    ↓
  Target Pod (10.244.0.5)
    ↓
  응답: 10.244.0.5:8080 → 10.244.0.10
  (자동으로 10.104.50.1으로 변환 역진행)
```

### 성능 특성

```
iptables 모드:
- Service 개수별 레이턴시
  * 100개 Service: 0.1ms
  * 500개 Service: 0.5ms
  * 1000개 Service: 1-2ms
  
- CPU 사용: Service 추가 시 CPU 스파이크

IPVS 모드:
- Service 개수 무관: 항상 0.05ms
- CPU 사용: 안정적, 선형 증가 없음

결론:
* 소규모 (<200 Services): iptables 충분
* 중규모 (200-1000): IPVS 권장
* 대규모 (>1000): IPVS 필수
```

---

## 🔵 4. 언제? (When)

### Service 선택 기준

```
ClusterIP
├─ 언제: Pod 간 통신만 필요
├─ 예: DB ← App ← Frontend
├─ 외부 접근 불필요
└─ 가장 안전 (기본값)

NodePort
├─ 언제: 외부 접근이 필요하지만 로드밸런서 없음
├─ 예: 온프레미스, 개발/테스트
├─ 포트 범위: 30000-32767
├─ 실제 production에서는 드물게 사용
└─ "임시" 외부 접근용

LoadBalancer
├─ 언제: 프로덕션 외부 서비스 (클라우드)
├─ 예: 웹 애플리케이션, API
├─ 클라우드 제공자의 로드밸런서 자동 생성
├─ AWS ELB / Azure LB / GCP LB
└─ 가장 완벽한 솔루션

ExternalName
├─ 언제: 외부 서비스를 DNS alias로 노출
├─ 예: 기존 데이터베이스 통합
├─ 실제 라우팅 없음 (DNS만)
└─ 마이그레이션 시나리오
```

### kube-proxy 모드 선택

```
선택 기준:

규모가 작음 (< 300 Services)
  → iptables (관리 복잡도 낮음)

규모가 중간 (300-1000 Services)
  → IPVS 권장 시작점

규모가 큼 (> 1000 Services)
  → IPVS 필수

성능이 중요
  → IPVS (항상)

안정성이 최고 우선
  → iptables (더 오래되고 검증됨)

클라우드 네이티브 (AKS)
  → 기본값 확인
  → 대부분 iptables (충분함)
  → 고성능 필요시 IPVS로 변경
```

---

## 💜 5. 실전 (Practice)

### Service 생성 및 동작 확인

```bash
# Deployment 생성
kubectl create deployment nginx --image=nginx --replicas=3

# Service 생성
kubectl expose deployment nginx --port=80 --target-port=8080

# Service 정보 확인
kubectl get svc nginx
# NAME    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# nginx   ClusterIP   10.104.50.1     <none>        80/TCP

# Endpoints 확인 (실제 Pod들)
kubectl get endpoints nginx
# NAME    ENDPOINTS
# nginx   10.244.0.5:8080,10.244.1.6:8080,10.244.2.7:8080

# Pod에서 Service로 접근 테스트
kubectl run -it debug --image=ubuntu -- bash
# curl http://nginx:80
# curl http://10.104.50.1:80
```

### kube-proxy 모드 확인 및 변경

```bash
# 현재 모드 확인
kubectl get daemonset -n kube-system kube-proxy -o yaml | grep mode:
# mode: iptables (기본)

# 모드 변경 (iptables → IPVS)
kubectl patch daemonset -n kube-system kube-proxy -p '{"spec":{"template":{"spec":{"containers":[{"name":"kube-proxy","env":[{"name":"KUBEPROXY_MODE","value":"ipvs"}]}]}}}}'

# 또는 configmap 수정
kubectl edit configmap -n kube-system kube-proxy
# mode: "" → mode: "ipvs"
```

### iptables 규칙 직접 확인

```bash
# Node에 접근
kubectl debug node/node1 -it --image=ubuntu

# KUBE- prefix 규칙 확인
iptables -L -t nat | grep KUBE

# 특정 Service IP의 규칙
iptables -L -t nat -n | grep 10.104.50.1

# Conntrack 확인 (활성 연결)
conntrack -L | grep 10.104.50.1
```

### 성능 테스트

```bash
# Service 대역폭 측정
kubectl exec -it pod1 -- iperf3 -s &
kubectl exec -it pod2 -- iperf3 -c nginx -t 10

# Service 레이턴시 (iptables)
kubectl exec -it pod1 -- ping -c 100 nginx | grep avg

# Service 레이턴시 (IPVS)
# IPVS로 변경 후 다시 측정
```

---

**다음**: Service 패킷 흐름 분석 📊
