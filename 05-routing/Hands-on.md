# 05: Routing 실습

## 🎯 목표
- 라우팅 테이블 조회 및 수정
- 다중 Node 간 라우팅 시뮬레이션
- CIDR와 최장 프리픽스 매칭

## 🔬 실습 1: 라우팅 테이블

```bash
# 현재 라우팅 테이블
ip route show

# 더 자세히
ip -4 route show
ip -6 route show

# 특정 대상의 라우트
ip route get 8.8.8.8
```

## 🔬 실습 2: 라우트 추가/삭제

```bash
# 라우트 추가
sudo ip route add 10.50.0.0/16 via 192.168.1.1 dev eth0

# 기본 게이트웨이 설정
sudo ip route add default via 192.168.1.1 dev eth0

# 라우트 삭제
sudo ip route del 10.50.0.0/16 via 192.168.1.1 dev eth0

# 확인
ip route show
```

## 🔬 실습 3: 다중 Namespace 라우팅

```bash
# 3개 Namespace 생성 (3개 노드 시뮬레이션)
sudo ip netns add node1
sudo ip netns add node2
sudo ip netns add node3

# Bridge 3개 생성
sudo ip link add br1 type bridge
sudo ip link add br2 type bridge
sudo ip link add br3 type bridge

sudo ip link set br1 up
sudo ip link set br2 up
sudo ip link set br3 up

# IP 할당 (각 node의 Pod CIDR)
sudo ip addr add 10.244.0.1/24 dev br1
sudo ip addr add 10.244.1.1/24 dev br2
sudo ip addr add 10.244.2.1/24 dev br3

# 각 namespace에서 bridge로 veth 연결
# ... (복잡, 문서만 참조)

# node1의 라우팅 테이블
sudo ip netns exec node1 ip route show
sudo ip netns exec node1 ip route add 10.244.1.0/24 via 10.240.0.2
sudo ip netns exec node1 ip route add 10.244.2.0/24 via 10.240.0.3

# 통신 테스트
sudo ip netns exec node1 ping -c 1 10.244.1.5
```

## 🔬 실습 4: 최장 프리픽스 매칭

```bash
# 여러 라우트 추가
sudo ip route add 10.0.0.0/8 via 192.168.1.1
sudo ip route add 10.244.0.0/16 via 192.168.1.2
sudo ip route add 10.244.1.0/24 via 192.168.1.3

# 어느 라우트가 사용될까?
ip route get 10.244.1.5
# 예상: 10.244.1.0/24이 최장 프리픽스 매치

ip route get 10.1.0.1
# 예상: 10.0.0.0/8이 사용됨
```

---

**다음**: iptables 실습 🔗
