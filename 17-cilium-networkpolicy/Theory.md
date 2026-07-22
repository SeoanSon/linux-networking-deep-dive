# 17: Cilium NetworkPolicy 이론

## 🎯 핵심

Cilium은 표준 NetworkPolicy뿐만 아니라 L7 정책도 지원합니다.

## 📖 1. CiliumNetworkPolicy (CNP)

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-http-get
spec:
  podSelector:
    matchLabels:
      app: web
  ingress:
  - fromEndpoints:
    - matchLabels:
        app: client
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
      rules:
        http:
        - method: "GET"
          path: "/api/v1/.*"  # 정규식
```

## 📖 2. L7 규칙 예

```
표준 NetworkPolicy (L4):
└─ Port 80 허용 (모든 HTTP 메서드)

Cilium L7 정책:
└─ Port 80 + GET + /api/v1/* 경로만 허용
└─ POST /admin/* 차단
└─ PUT 전부 차단

작동:
App → /api/v1/users (GET) → 허용
App → /api/v1/admin (POST) → 차단
App → /admin (PUT) → 차단
```

---

**다음: Observability** 📊
