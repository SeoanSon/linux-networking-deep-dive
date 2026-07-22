# 06: iptables 실습

## 🎯 목표
- iptables 규칙 조회 및 설정
- 필터링 및 NAT 실습

## 🔬 실습 1: 기본 명령어

```bash
# 현재 규칙 확인
sudo iptables -L

# 더 자세히
sudo iptables -L -n -v

# 특정 테이블
sudo iptables -t nat -L

# 기본정책 확인
sudo iptables -L INPUT
```

## 🔬 실습 2: 필터링 규칙

```bash
# SSH 포트 허용
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 특정 IP 차단
sudo iptables -A INPUT -s 192.168.1.100 -j DROP

# 모든 inbound 허용 (위험!)
sudo iptables -P INPUT ACCEPT

# 규칙 제거
sudo iptables -D INPUT -s 192.168.1.100 -j DROP

# 모든 규칙 초기화
sudo iptables -F
```

## 🔬 실습 3: NAT

```bash
# SNAT: 출발지 IP 변환
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/8 -j MASQUERADE

# DNAT: 목적지 IP 변환
sudo iptables -t nat -A PREROUTING -p tcp -d 203.0.113.1 --dport 80 -j DNAT --to-destination 10.0.0.1:8080

# 저장 (CentOS/RHEL)
sudo service iptables save

# Ubuntu
sudo apt install iptables-persistent
sudo iptables-save > /etc/iptables/rules.v4
```

---

**다음**: netfilter 실습 🔗
