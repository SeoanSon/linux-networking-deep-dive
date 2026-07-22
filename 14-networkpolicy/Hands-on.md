# 14: NetworkPolicy 실습

## 🎯 목표
- NetworkPolicy 생성 및 적용
- 트래픽 제어 확인

## 🔬 실습 1: Deny All

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

적용 및 테스트:
```bash
kubectl apply -f deny-all.yaml

# 통신 차단 확인
kubectl exec pod1 -- curl http://pod2  # 실패
```

## 🔬 실습 2: Selective Allow

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: client
    ports:
    - protocol: TCP
      port: 80
```

적용:
```bash
# Pod에 라벨 추가
kubectl label pod pod1 app=web
kubectl label pod pod2 role=client

# 정책 적용
kubectl apply -f allow-web.yaml

# 통신 확인
kubectl exec pod2 -- curl http://pod1  # 허용
```

## 🔬 실습 3: 정책 확인

```bash
kubectl get networkpolicies
kubectl describe networkpolicy allow-web
```

---

**다음**: eBPF 실습 🔗
