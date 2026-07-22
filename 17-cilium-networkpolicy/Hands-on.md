# 17: Cilium NetworkPolicy 실습

## 🎯 목표
- L7 정책 작성 및 적용
- HTTP 기반 필터링

## 🔬 실습 1: CiliumNetworkPolicy 작성

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-policy
spec:
  podSelector:
    matchLabels:
      app: api
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: client
    toPorts:
    - ports:
      - port: "8080"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"
```

적용 및 테스트:
```bash
kubectl apply -f l7-policy.yaml

# GET /api/v1/* 허용, 나머지는 차단
kubectl exec client -- curl http://api:8080/api/v1/users  # 허용
kubectl exec client -- curl http://api:8080/admin  # 차단
```

---

**다음**: Observability 실습 🔗
