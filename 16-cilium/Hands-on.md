# 16: Cilium 실습

## 🎯 목표
- Cilium 설치 및 기본 사용
- eBPF 기반 네트워킹 체험

## 🔬 실습 1: Cilium 설치

```bash
# Helm repo 추가
helm repo add cilium https://helm.cilium.io
helm repo update

# Cilium 설치
helm install cilium cilium/cilium --namespace kube-system

# 확인
kubectl get pods -n kube-system -l k8s-app=cilium
```

## 🔬 실습 2: Hubble 모니터링

```bash
# Hubble 설치
helm install hubble cilium/cilium --namespace kube-system --set hubble.enabled=true

# Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 8081:80

# 브라우저: http://localhost:8081
```

---

**다음**: Cilium NetworkPolicy 실습 🔗
