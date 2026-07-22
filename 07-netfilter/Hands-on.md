# 07: netfilter 실습

## 🎯 목표
- netfilter 통계 확인
- Hook points 이해

## 🔬 실습 1: netfilter 통계

```bash
# netfilter 규칙 통계
sudo cat /proc/net/netfilter/nfnetlink_log

# conntrack 확인
sudo cat /proc/net/nf_conntrack | head -10

# conntrack 한계
cat /proc/sys/net/nf_conntrack_max

# 현재 연결 수
cat /proc/sys/net/netfilter/nf_conntrack_count
```

## 🔬 실습 2: conntrack 모니터링

```bash
# 실시간 모니터링
sudo conntrack -E

# 특정 프로토콜만
sudo conntrack -E -p tcp

# 저장된 연결 확인
sudo conntrack -L
```

---

**다음**: Containers 실습 🔗
