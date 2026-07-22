# 15: eBPF (Extended Berkeley Packet Filter) 이론

## 🎯 핵심

**eBPF**는 Linux 커널에서 안전하게 프로그램을 실행할 수 있는 기술입니다.

## 📖 1. eBPF vs iptables

```
iptables:
- 사용자공간에서 규칙 생성
- 커널이 규칙 코드 생성
- 새 규칙마다 커널 코드 재컴파일 (오버헤드)
- 기능 제한 (L4까지만)

eBPF:
- 프로그램을 바이트코드로 작성
- 커널에서 JIT 컴파일
- 즉시 로드 (업데이트 빠름)
- L7 프로토콜까지 지원 가능

성능:
eBPF: ⭐⭐⭐⭐⭐ (매우 빠름, 커널 메모리)
iptables: ⭐⭐⭐ (느림, 규칙이 많을수록)
```

## 📖 2. eBPF Hook Points

```
네트워킹 관련 Hook:

tc (Traffic Control):
  ingress hook ← 패킷 도착
  egress hook ← 패킷 송신

sk_skb (Socket SKB):
  TCP 소켓 처리

xdp (eXpress Data Path):
  드라이버 직후 (최고 성능)

kprobes/tracepoints:
  커널 추적 및 모니터링
```

---

**다음: Cilium** 🌀
