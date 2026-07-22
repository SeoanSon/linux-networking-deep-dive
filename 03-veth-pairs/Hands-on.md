# 03: veth Pairs 실습

## 🎯 목표
- veth 쌍 생성 및 테스트
- 양방향 통신 확인

## 🔬 실습 1: veth 생성

```bash
# veth 쌍 생성
sudo ip link add veth_host type veth peer name veth_ns

# 확인
ip link show | grep veth

# IP 할당
sudo ip addr add 10.100.1.1/24 dev veth_host
sudo ip link set veth_host up

# Namespace에 이동
sudo ip netns add container_ns
sudo ip link set veth_ns netns container_ns

# Namespace에서 IP 할당
sudo ip netns exec container_ns ip addr add 10.100.1.2/24 dev veth_ns
sudo ip netns exec container_ns ip link set veth_ns up

# 통신 테스트
sudo ip netns exec container_ns ping -c 1 10.100.1.1
```

## 🔬 실습 2: 성능 측정

```bash
# iperf를 이용한 대역폭 측정
# (설치: sudo apt install iperf3)

# Namespace에서 서버
sudo ip netns exec container_ns iperf3 -s

# 호스트에서 클라이언트
iperf3 -c 10.100.1.2
```

---

**다음**: Linux Bridge 실습 🔗
