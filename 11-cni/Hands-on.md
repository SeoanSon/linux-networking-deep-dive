# 11: CNI 실습

## 🎯 목표
- CNI 플러그인 조회
- Pod 네트워킹 구성 확인

## 🔬 실습 1: CNI 설정 확인

```bash
# CNI 설정 파일
ls /etc/cni/net.d/

# 내용 확인
cat /etc/cni/net.d/10-flannel.conflist

# CNI 바이너리
ls /opt/cni/bin/
```

## 🔬 실습 2: CNI 플러그인 별 특성

```bash
# Flannel 확인
kubectl get daemonset -n kube-system

# Calico (설치된 경우)
kubectl get daemonset -n kube-system calico-node

# 각 Node의 CNI
kubectl debug node/node-name -it --image=ubuntu
# ls /etc/cni/net.d/
```

---

**다음**: kube-proxy 실습 🔗
