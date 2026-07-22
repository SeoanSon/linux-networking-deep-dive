# 02: Network Namespace 실습

## 🎯 목표
- Network Namespace 생성 및 관리
- 네트워크 격리 확인

## 📋 준비
- Linux (WSL2, Ubuntu)
- sudo 권한
- ip, ping 명령어

## 🔬 실습 1: Namespace 생성

```bash
# 새로운 Network Namespace 생성
sudo ip netns add test_ns

# Namespace 확인
ip netns list

# Namespace 내에서 명령 실행
sudo ip netns exec test_ns ip addr

# 다른 작업: 호스트
ip addr  # 호스트의 인터페이스
```

## 🔬 실습 2: Namespace 간 통신

```bash
# 2개 namespace 생성
sudo ip netns add ns1
sudo ip netns add ns2

# veth 쌍 생성 (다음 장에서 상세히)
sudo ip link add veth1 type veth peer name veth2

# namespace로 이동
sudo ip link set veth1 netns ns1
sudo ip link set veth2 netns ns2

# IP 할당
sudo ip netns exec ns1 ip addr add 10.0.0.1/24 dev veth1
sudo ip netns exec ns2 ip addr add 10.0.0.2/24 dev veth2

# 인터페이스 활성화
sudo ip netns exec ns1 ip link set veth1 up
sudo ip netns exec ns2 ip link set veth2 up

# 통신 테스트
sudo ip netns exec ns1 ping -c 1 10.0.0.2
```

## 🔬 실습 3: Namespace 정리

```bash
# namespace 삭제
sudo ip netns delete ns1
sudo ip netns delete ns2

# 확인
ip netns list
```

---

**다음**: veth Pairs 실습 🔗
