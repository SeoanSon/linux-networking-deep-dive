# 11: CNI (Container Network Interface) 이론

## 🎯 핵심

CNI는 Kubernetes와 네트워크 플러그인 사이의 표준 인터페이스입니다.

## 📖 1. CNI 동작

```
kubelet이 Pod 생성:
1. containerd에 요청
2. containerd가 Namespace 생성
3. kubelet이 CNI 플러그인 호출
   $ /opt/cni/bin/flannel add <container_id>
4. CNI가 veth 생성, IP 할당, 라우팅 설정
5. Pod 준비 완료

CNI 구성 파일:
/etc/cni/net.d/10-flannel.conflist
├─ 플러그인 설정
├─ IPAM (IP할당 메커니즘)
└─ Pod CIDR 범위
```

## 📖 2. CNI 플러그인 비교

```
Flannel:
├─ UDP 기반 터널링
├─ 간단, 느림
└─ 소규모 클러스터용

Calico:
├─ BGP 기반 라우팅
├─ 성능 좋음, NetworkPolicy 지원
└─ 중대형 클러스터용

Cilium:
├─ eBPF 기반
├─ L7 policy, 보안, 성능
└─ 모던 클러스터용
```

---

**다음: kube-proxy** 🎯
