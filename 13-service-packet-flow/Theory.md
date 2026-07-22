# 13: Service Packet Flow 이론

## 🎯 핵심

Service로의 패킷 흐름을 완벽히 추적합니다.

## 📖 1. 완전한 흐름

```
Client Pod (10.244.0.5) → Service (10.104.50.1:80) → Server Pod (10.244.1.7:8080)

Step 1: Client에서 패킷 생성
SrcIP: 10.244.0.5, DstIP: 10.104.50.1, Port: 80

Step 2: Node 1에서 라우팅 결정
라우팅 테이블: 10.104.50.1 = 로컬 Service
└─ Node 내에서 처리

Step 3: kube-proxy iptables 규칙 적용
PREROUTING: 10.104.50.1:80 → 10.244.1.7:8080으로 DNAT
└─ 패킷 변환: DstIP=10.244.1.7, DstPort=8080

Step 4: 라우팅
라우팅 테이블: 10.244.1.0/24 → Node 2로
└─ eth0으로 Node 2로 전송

Step 5: Node 2에서 수신
라우팅: 10.244.1.7 = 로컬 Pod
└─ cbr0 → veth → Pod로 전송

Step 6: Server Pod에서 수신
App이 8080에서 수신, 처리

Step 7: Server에서 응답
SrcIP: 10.244.1.7:8080, DstIP: 10.244.0.5

Step 8: Node 2에서 역 NAT (자동)
POSTROUTING: Conntrack에서 원본 변환 제거
└─ DstIP: 10.244.0.5 (Client)

Step 9: 다시 Node 1로 라우팅
라우팅 테이블: 10.244.0.0/24 → Node 1

Step 10: Client Pod가 수신
앱이 응답 패킷 수신
└─ SrcIP: 10.104.50.1 (Service IP)
└─ 마치 Service에서 온 것처럼 보임
```

## 📖 2. Session Affinity

```
같은 Client의 패킷을 같은 Pod로:

Service 정의:
sessionAffinity: ClientIP
sessionAffinityConfig:
  clientIPConfig:
    timeoutSeconds: 10800

작동:
첫 번째 패킷: 10.244.0.5 → Pod A 선택
10시간 동안: 10.244.0.5의 모든 패킷 → Pod A

구현: iptables에서 Client IP를 해시하여 같은 Pod 선택
```

---

**다음: NetworkPolicy** 🔒
