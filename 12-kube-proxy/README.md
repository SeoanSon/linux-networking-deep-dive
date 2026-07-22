# 12: kube-proxy (Service의 실제 구현)

> **목표**: Kubernetes Service가 내부적으로 어떻게 작동하는가
>
> **학습 시간**: 4.5시간
>
> **난이도**: ⭐⭐⭐ 중급-고급

## 🎯 핵심

- kube-proxy의 3가지 모드: userspace, iptables, ipvs
- ClusterIP, NodePort, LoadBalancer 구현
- Endpoint와 DNAT (Destination NAT)
- Service 트래픽 분산 메커니즘

## ➡️ 다음: 13 Service Packet Flow
