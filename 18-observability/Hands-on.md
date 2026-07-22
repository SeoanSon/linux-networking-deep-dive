# 18: Observability 실습

## 🎯 목표
- 네트워킹 문제 진단
- 패킷 흐름 모니터링

## 🔬 실습 1: kubectl debug

```bash
# Pod 디버그
kubectl debug pod/mypod -it --image=ubuntu

# Node 디버그
kubectl debug node/mynode -it --image=ubuntu
```

## 🔬 실습 2: 로그 분석

```bash
# kubelet 로그
kubectl logs -n kube-system kubelet-... | grep error

# CNI 로그
kubectl logs -n kube-system -l k8s-app=flannel | tail -20
```

## 🔬 실습 3: 메트릭 수집

```bash
# Prometheus 설치
helm install prometheus prometheus-community/kube-prometheus-stack

# 메트릭 쿼리
kubectl port-forward -n default svc/prometheus 9090:9090
# http://localhost:9090
```

---

**다음**: Real-world AKS 실습 🔗
