# 12: kube-proxy 이론

## 🎯 핵심

**kube-proxy**는 Kubernetes Service를 구현하는 Node agent입니다.

## 📖 1. kube-proxy 역할

```
Service 생성:
apiVersion: v1
kind: Service
metadata:
  name: nginx
spec:
  clusterIP: 10.104.50.1
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: nginx

kube-proxy가 처리:
1. Service IP 감지 (10.104.50.1)
2. Endpoints 감지 (Pod IP들: 10.244.0.5, 10.244.0.6)
3. iptables/IPVS 규칙 생성
   10.104.50.1:80 → 10.244.0.5:8080 또는 10.244.0.6:8080

작동:
Pod/Client → Service IP:Port
          ↓
        kube-proxy 규칙
          ↓
        Pod IP:Port로 변환 및 라우팅
```

## 📖 2. kube-proxy 모드

```
1️⃣ iptables 모드 (기본)
   └─ iptables 규칙으로 구현
   └─ 느린 업데이트 (Linux 커널 코드 생성)

2️⃣ IPVS 모드 (고성능)
   └─ Linux IPVS (IP Virtual Server) 사용
   └─ 빠른 업데이트
   └─ 로드밸런싱 알고리즘 다양 (RR, LC 등)

3️⃣ userspace 모드 (레거시)
   └─ 사용자공간 구현
   └─ 느림, 일반적으로 사용 안 함

선택:
$ kubectl edit daemonset -n kube-system kube-proxy
ipvs로 변경 가능
```

---

**다음: Service Packet Flow** 🎯
