# 19: Real-world AKS Networking 실습

## 🎯 목표
- AKS 클러스터 생성 및 네트워킹 설정
- 프로덕션 구성 체험

## 🔬 실습 1: AKS 클러스터 생성

```bash
# 변수 설정
RG="myResourceGroup"
CLUSTER_NAME="myCluster"
LOCATION="eastus"

# 리소스 그룹
az group create -n $RG -l $LOCATION

# AKS 클러스터 (Azure CNI)
az aks create \
  -n $CLUSTER_NAME \
  -g $RG \
  --network-plugin azure \
  --network-policy calico \
  --zones 1 2 3 \
  -k 1.28
```

## 🔬 실습 2: 네트워킹 검증

```bash
# Credentials 가져오기
az aks get-credentials -n $CLUSTER_NAME -g $RG

# 클러스터 확인
kubectl get nodes -o wide

# Pod 네트워킹 확인
kubectl run test --image=nginx
kubectl get pods -o wide
```

## 🔬 실습 3: LoadBalancer 서비스

```bash
# LoadBalancer Service
kubectl expose pod test --port=80 --type=LoadBalancer

# Public IP 확인
kubectl get svc -w

# 외부 접속 테스트
curl http://<EXTERNAL_IP>
```

## 🔬 실습 4: Ingress 설정

```bash
# Nginx Ingress Controller 설치
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress ingress-nginx/ingress-nginx

# Ingress 생성
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test
            port:
              number: 80
```

## 🔬 실습 5: 정리

```bash
# 클러스터 삭제
az aks delete -n $CLUSTER_NAME -g $RG --yes

# 리소스 그룹 삭제
az group delete -n $RG --yes
```

---

## 🎓 축하합니다!

**모든 19장의 실습을 완료했습니다!** 🎉

다음 단계:
- Phase 3: Architecture 다이어그램
- Phase 4: PacketFlow 상세 분석
- Phase 5: Troubleshooting 시나리오
- Phase 6: 면접 대비 질문
