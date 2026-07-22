# 10: Kubernetes Networking 실습

## 🎯 목표
- Pod IP 할당 확인
- Pod 간 통신 테스트

## 🔬 실습 1: Pod IP 확인

```bash
# Pod 생성
kubectl run pod1 --image=nginx --port=80
kubectl run pod2 --image=nginx --port=80
kubectl run pod3 --image=busybox --image-pull-policy=IfNotPresent -- sleep 3600

# Pod IP 확인
kubectl get pods -o wide

# Pod 상세 정보
kubectl describe pod pod1
```

## 🔬 실습 2: Pod 간 통신

```bash
# pod1에서 pod2로 ping
kubectl exec pod1 -- ping -c 1 <pod2_IP>

# 또는 pod3에서
kubectl exec pod3 -- ping -c 1 <pod1_IP>

# Service를 통한 통신
kubectl expose pod pod1 --port=80 --name=pod1-svc
kubectl exec pod2 -- curl http://pod1-svc
```

## 🔬 실습 3: Node 네트워킹

```bash
# Node 정보
kubectl get nodes -o wide

# Node의 cbr0 확인 (각 Node에 SSH)
# 또는 debug pod 사용
kubectl debug node/node-name -it --image=ubuntu

# Node 내에서
ip addr show cbr0
ip route show
```

---

**다음**: CNI 실습 🔗
