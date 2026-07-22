# 18: Observability (관찰성) 이론

## 🎯 핵심

네트워킹 문제를 진단하기 위해 패킷 흐름과 성능을 관찰합니다.

## 📖 1. Observability 도구

```
1️⃣ tcpdump (패킷 캡처)
   $ tcpdump -i eth0 -nn host 10.244.0.5
   └─ 실시간 패킷 보기

2️⃣ Cilium Hubble (시각화)
   $ cilium hubble ui
   └─ Pod 간 트래픽을 그래프로 표시

3️⃣ kubectl logs (Pod 로그)
   $ kubectl logs pod/myapp

4️⃣ kubectl debug (Node 접근)
   $ kubectl debug node/mynode -it --image=ubuntu
```

## 📖 2. 진단 흐름

```
Pod 간 통신 안 됨?

1️⃣ Pod IP 확인
   $ kubectl get pods -o wide

2️⃣ Service 확인
   $ kubectl get svc
   $ kubectl get endpoints

3️⃣ 라우팅 확인
   $ kubectl debug node/node1
   # ip route show

4️⃣ 방화벽 확인
   # iptables -L (iptables 모드)
   # cilium policy get (Cilium 모드)

5️⃣ 패킷 캡처
   # tcpdump -i veth... icmp
   # ping 대상으로 테스트

6️⃣ NetworkPolicy 확인
   $ kubectl get networkpolicies
```

---

**다음: Real-world AKS** 🏗️
