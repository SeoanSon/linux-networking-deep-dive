# 12: kube-proxy 실습

## 🎯 목표
- kube-proxy 동작 확인
- iptables/IPVS 규칙 확인

## 🔬 실습 1: kube-proxy 모드 확인

```bash
# kube-proxy 상태
kubectl get daemonset -n kube-system kube-proxy

# 설정 확인
kubectl get configmap -n kube-system kube-proxy-config -o yaml

# 로그 확인
kubectl logs -n kube-system -l k8s-app=kube-proxy | head -20
```

## 🔬 실습 2: Service와 Endpoints

```bash
# Service 생성
kubectl create service clusterip nginx --tcp=80:80

# Endpoints 확인
kubectl get endpoints nginx

# Service 상세정보
kubectl describe svc nginx
```

## 🔬 실습 3: iptables 규칙 확인

```bash
# Node에 접근
kubectl debug node/node-name -it --image=ubuntu

# iptables 규칙 확인
sudo iptables -t nat -L | grep nginx

# IPVS 모드인 경우
ipvsadm -L
```

---

**다음**: Service Packet Flow 실습 🔗
