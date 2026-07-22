# 09: Container Runtime Networking 실습

## 🎯 목표
- containerd 네트워킹 이해
- CRI 통신 확인

## 🔬 실습 1: containerd 정보

```bash
# containerd 설치
sudo apt install containerd.io

# containerd CLI
sudo ctr ns list

# 이미지 다운로드
sudo ctr images pull docker.io/library/nginx:latest

# 컨테이너 실행
sudo ctr run --net-host docker.io/library/nginx:latest mynginx

# 네트워크 namespace 확인
ps aux | grep containerd
```

## 🔬 실습 2: Kubernetes 준비

```bash
# minikube 또는 kind로 테스트
# minikube
minikube start

# kind
kind create cluster

# pod 생성
kubectl run nginx --image=nginx
kubectl get pods -o wide
```

---

**다음**: Kubernetes Networking 실습 🔗
