# 19: Real-world AKS Networking 이론

## 🎯 핵심

Azure Kubernetes Service (AKS)는 Kubernetes를 Azure의 특성에 맞게 구현합니다.

## 📖 1. AKS 네트워크 아키텍처

```
AKS Cluster (Azure VNet):
┌──────────────────────────────────┐
│  VNet: 10.240.0.0/12             │
│  (노드 서브넷)                    │
│                                  │
│  Node 1: 10.240.0.5              │
│  ├─ eth0: 10.240.0.5             │
│  ├─ cbr0: 10.244.0.1             │
│  └─ Pod들: 10.244.0.0/24         │
│                                  │
│  Node 2: 10.240.0.6              │
│  ├─ eth0: 10.240.0.6             │
│  ├─ cbr0: 10.244.1.1             │
│  └─ Pod들: 10.244.1.0/24         │
└──────────────────────────────────┘
       ↓
  Azure Virtual Network
  └─ NSG (Network Security Group)
```

## 📖 2. AKS의 특별한 점

```
표준 Kubernetes:
- CNI 플러그인 선택 필요
- 네트워킹 직접 관리

AKS:
- Azure CNI 또는 Kubenet 선택 가능
- NSG로 인바운드/아웃바운드 제어
- Application Gateway 통합
- Service Principal으로 Azure 리소스 접근
```

## 📖 3. AKS 네트워킹 옵션

### Azure CNI

```yaml
# AKS 클러스터 생성 시
az aks create \
  --resource-group myRG \
  --name myCluster \
  --network-plugin azure \
  --service-cidr 10.104.0.0/12 \
  --dns-service-ip 10.104.0.10 \
  --docker-bridge-address 172.17.0.1/16 \
  --vnet-subnet-id /subscriptions/.../subnets/pods

특징:
- Pod가 VNet 직접 사용
- 고성능
- Pod 수 제한 (Node당 ~30개)
```

### Kubenet

```yaml
# AKS 클러스터 생성 시
az aks create \
  --resource-group myRG \
  --name myCluster \
  --network-plugin kubenet \
  --service-cidr 10.104.0.0/12 \
  --dns-service-ip 10.104.0.10 \
  --pod-cidr 10.244.0.0/12

특징:
- Pod는 NAT 뒤에 (내부 네트워크)
- 안전 (외부 직접 접근 불가)
- Pod 수 제한 없음
- 느린 클러스터 간 통신
```

## 📖 4. LoadBalancer와 Ingress

```
외부 클라이언트 → Azure Load Balancer
                ↓
             Public IP (20.x.x.x)
                ↓
         Service (LoadBalancer)
                ↓
            Pods

또는:

외부 클라이언트 → Application Gateway
                ↓
             Public IP (20.x.x.x)
                ↓
         Ingress (Nginx/APIM)
                ↓
             Service
                ↓
            Pods
```

## 📖 5. 보안 (NSG/NetworkPolicy)

```
AKS의 3단계 보안:

1️⃣ NSG (Azure 수준)
   └─ VNet 레벨에서 트래픽 제어
   └─ 노드의 인바운드/아웃바운드

2️⃣ NetworkPolicy (Kubernetes 수준)
   └─ Pod 간 통신 제어
   └─ CNI 플러그인(Calico/Cilium)이 구현

3️⃣ Pod Security Policy / Standards (Pod 레벨)
   └─ 컨테이너 권한, 마운트, 리소스 제어

예시:
NSG: "10.240.0.0/24 노드만 22번 접근 허용"
NetworkPolicy: "nginx Pod만 80번 수신"
PSP: "모든 Pod은 root로 실행 불가"
```

## 📖 6. Private Cluster

```
AKS Private Cluster:

클라이언트 → Azure Bastion
         → 점프박스(Bastion)
         → Private Endpoint
         → API Server (비공개)

kubectl 접근:
1. Bastion에 접근
2. kubectl config 설정
3. Private API Server와 통신

네트워킹:
- API Server: Public IP 없음
- 클러스터 외부에서 직접 접근 불가
- ExpressRoute/VPN을 통해서만 접근
```

## 📖 7. 실제 구성 예

```yaml
# AKS 프로덕션 권장 구성
az aks create \
  --resource-group production \
  --name prod-cluster \
  --network-plugin azure \
  --network-policy calico \      # Calico로 NetworkPolicy 지원
  --zones 1 2 3                  # 3개 Availability Zone
  --load-balancer-sku standard \
  --vm-set-type VirtualMachineScaleSets \
  --enable-managed-identity \
  --api-server-authorized-ip-ranges 10.0.0.0/8

결과:
✅ Pod 고성능 (Azure CNI)
✅ 네트워크 정책 (Calico)
✅ 고가용성 (3개 AZ)
✅ 보안 (관리 ID, IP 제한)
```

---

## 🎓 결론

**Linux Networking의 여정:**
- Process & IPC (01)
- Network Namespace (02)
- veth 쌍 (03)
- Linux Bridge (04)
- Routing (05)
- Filtering (06)
- Kernel Engine (07)
- Containers (08-09)
- Kubernetes 통합 (10)
- CNI (11)
- Service (12-13)
- Policy (14-17)
- Observability (18)
- Real-world AKS (19)

**다음 학습:**
- Phase 2: 각 장별 Hands-on 실습
- Phase 3: 아키텍처 다이어그램
- Phase 4: 패킷 흐름 트레이싱
- Phase 5: 트러블슈팅 시나리오
- Phase 6: 면접 질문 대비

**감사합니다!** 🙏
