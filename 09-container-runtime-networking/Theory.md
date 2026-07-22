# 09: Container Runtime Networking 이론

## 🎯 핵심

Container Runtime (Docker, containerd, CRI-O)이 컨테이너의 네트워킹을 초기화하고 관리합니다.

## 📖 1. Runtime의 역할

```
1️⃣ Network Namespace 생성
   └─ clone(CLONE_NEWNET)

2️⃣ 네트워크 구성
   └─ Host 또는 CNI에 위임

3️⃣ 컨테이너 실행
   └─ Namespace 내에서 프로세스 시작

4️⃣ 정리
   └─ 종료 시 Namespace 정리
```

## 📖 2. Docker vs containerd

### Docker
```
User: docker run myimage
    ↓
Docker Daemon
    ├─ Image 준비
    ├─ Container 생성
    ├─ Network 설정 (docker0 bridge)
    ├─ 프로세스 시작
    └─ 모니터링
```

### containerd (Kubernetes 표준)
```
User: ctr run myimage
    ↓
containerd
    ├─ Image 준비
    ├─ Container 생성
    ├─ Network은 CNI 플러그인에 위임
    ├─ 프로세스 시작
    └─ 모니터링
```

## 📖 3. CRI (Container Runtime Interface)

```
Kubernetes는 CNI를 통해 runtime과 통신:

kubelet (Node agent)
    ↓
CRI (StandardInterface)
    ├─ CreateContainer
    ├─ StartContainer
    ├─ StopContainer
    └─ DeleteContainer
    ↓
containerd/Docker
    ├─ Namespace 생성
    ├─ CNI 호출
    └─ 프로세스 실행
```

---

**다음: Kubernetes Networking 통합** 🎯
