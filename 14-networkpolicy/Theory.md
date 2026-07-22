# 14: NetworkPolicy 이론

## 🎯 핵심

**NetworkPolicy**는 Pod 간 통신을 제어하는 Kubernetes 리소스입니다.

## 📖 1. NetworkPolicy 예

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: client
    ports:
    - protocol: TCP
      port: 80
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: db
    ports:
    - protocol: TCP
      port: 5432
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53
```

## 📖 2. 작동 원리

```
Pod web이 정책 위반 패킷 수신 시도:

1. Pod client의 패킷: 80번으로 → ACCEPT
2. Pod other의 패킷: 80번으로 → DROP (정책 위반)
3. Pod web의 모든 egress: DENY (정책에 없음)
   └─ 정책에 명시된 것만 허용

구현:
- Calico: iptables 규칙
- Cilium: eBPF 프로그램
- Weave: 자체 필터링
```

## 📖 3. 기본 정책 (Default)

```
정책 없음:
모든 트래픽 허용 (Allow All)

Deny All Ingress:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress

Allow All Egress (기본):
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - {}  # 모든 egress 허용
```

---

**다음: eBPF** 🚀
