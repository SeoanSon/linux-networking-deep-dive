# 04: Linux Bridge 실습

## 🎯 목표
- Linux Bridge 생성
- 여러 veth를 Bridge로 연결
- MAC 학습 확인

## 🔬 실습 1: Bridge 생성

```bash
# Bridge 생성
sudo ip link add br0 type bridge

# Bridge에 IP 할당
sudo ip addr add 10.200.1.1/24 dev br0
sudo ip link set br0 up

# veth 2개 생성
sudo ip link add veth1 type veth peer name veth1_peer
sudo ip link add veth2 type veth peer name veth2_peer

# Bridge에 추가
sudo ip link set veth1 master br0
sudo ip link set veth2 master br0

# 활성화
sudo ip link set veth1 up
sudo ip link set veth2 up

# 확인
brctl show
```

## 🔬 실습 2: Namespace와 Bridge 연결

```bash
# 2개 Namespace 생성
sudo ip netns add ns1
sudo ip netns add ns2

# veth를 namespace로 이동
sudo ip link set veth1_peer netns ns1
sudo ip link set veth2_peer netns ns2

# IP 할당
sudo ip netns exec ns1 ip addr add 10.200.1.2/24 dev veth1_peer
sudo ip netns exec ns2 ip addr add 10.200.1.3/24 dev veth2_peer

# 활성화
sudo ip netns exec ns1 ip link set veth1_peer up
sudo ip netns exec ns2 ip link set veth2_peer up

# 통신 테스트
sudo ip netns exec ns1 ping -c 1 10.200.1.3

# MAC 테이블 확인
sudo brctl showmacs br0
```

## 🔬 실습 3: 정리

```bash
sudo ip link del br0
sudo ip netns del ns1
sudo ip netns del ns2
```

---

**다음**: Routing 실습 🔗
