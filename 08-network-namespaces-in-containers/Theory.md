# 08: Network Namespaces in Containers 이론

## 🎯 핵심

Docker/컨테이너가 Network Namespace를 사용하여 각 컨테이너에 독립적인 네트워크 스택을 제공합니다.

## 📖 1. Docker의 네트워크 구조

```
Host OS
├─ Host Namespace (eth0: 172.16.0.5)
│
├─ docker0 (Linux Bridge)
│  ├─ veth_container1
│  └─ veth_container2
│
├─ Container 1 Namespace
│  └─ eth0 (실제로는 veth, IP: 172.17.0.2)
│
└─ Container 2 Namespace
   └─ eth0 (실제로는 veth, IP: 172.17.0.3)

각 컨테이너:
├─ 자신의 Network Namespace 가짐
├─ eth0 인터페이스 (veth pair의 한쪽)
├─ lo (127.0.0.1)
└─ 독립적인 포트 공간 (:80, :8080 등)
```

## 📖 2. Docker 네트워크 드라이버

```
1️⃣ bridge (기본)
   └─ docker0을 통해 연결
   └─ Host와도 연결 가능 (포트포워딩)

2️⃣ host
   └─ Host Namespace 사용
   └─ 격리 없음, 최고 성능

3️⃣ none
   └─ Network Namespace만 있고 인터페이스 없음
   └─ 완전 격리

4️⃣ overlay (Docker Swarm)
   └─ 여러 호스트의 컨테이너 연결

5️⃣ macvlan
   └─ 각 컨테이너에 고유 MAC
```

## 📖 3. Container Runtime과 Network Namespace

```
컨테이너 생성 시:

1️⃣ Container Runtime (Docker, containerd)
   └─ clone(CLONE_NEWNET) 호출
   └─ 새 Network Namespace 생성

2️⃣ 컨테이너 프로세스 시작
   └─ 이 Namespace에서 실행

3️⃣ CNI 플러그인 호출 (Kubernetes인 경우)
   └─ veth 생성 및 연결
   └─ IP 할당
   └─ 라우팅 설정

결과:
각 컨테이너 = 자신의 네트워크 스택
```

---

## 🎓 핵심

- 각 컨테이너는 Network Namespace 가짐
- Docker bridge로 컨테이너들 연결
- 포트포워딩으로 외부 접근 제공
- Kubernetes CNI가 더 정교한 네트워킹 구현

**다음: Container Runtime - Docker, containerd 등** 🐋
