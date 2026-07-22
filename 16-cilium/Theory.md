# 16: Cilium 이론

## 🎯 핵심

**Cilium**은 eBPF 기반의 CNI 플러그인으로 NetworkPolicy를 L7까지 지원합니다.

## 📖 1. Cilium 아키텍처

```
Cilium을 사용한 클러스터:

각 Node:
  cilium-agent (Daemon)
    ├─ eBPF 프로그램 로드
    ├─ Pod IP 관리
    ├─ NetworkPolicy 구현
    └─ 트래픽 모니터링

중앙:
  Cilium Operator (선택사항)
    └─ etcd와 동기화
```

## 📖 2. Cilium의 특징

```
1️⃣ 성능
   └─ eBPF로 커널 직접 처리
   └─ iptables 없음 (더 빠름)

2️⃣ L7 Policies
   HTTP/gRPC 기반 정책 (경로/메서드별)

3️⃣ Observability
   Hubble로 트래픽 시각화 및 모니터링

4️⃣ Encryption
   mTLS를 자동으로 관리 (Tetragon)
```

## 📖 3. Cilium 설치 (AKS)

```bash
# Cilium 헬름 차트로 설치
helm repo add cilium https://helm.cilium.io
helm install cilium cilium/cilium --namespace kube-system

# 확인
kubectl get pods -n kube-system | grep cilium

# Hubble 확인
cilium hubble ui
```

---

**다음: Cilium NetworkPolicy** 🔐
