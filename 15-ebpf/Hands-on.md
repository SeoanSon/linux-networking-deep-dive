# 15: eBPF 실습

## 🎯 목표
- eBPF 프로그램 기본 작성
- 커널 모니터링

## 🔬 실습 1: BCC로 eBPF 스크립트 작성

```bash
# BCC 설치
sudo apt install bpfcc-tools

# 시스템 콜 추적
sudo /usr/sbin/opensnoop

# 네트워킹 모니터링
sudo /usr/sbin/tcpconnect

# 패킷 필터링
sudo /usr/sbin/tcpdrop
```

## 🔬 실습 2: 간단한 eBPF 프로그램 (Python BCC)

```python
from bcc import BPF

# eBPF 프로그램 (C 코드)
prog = """
int trace_syscalls(struct pt_regs *ctx) {
    bpf_trace_printk("syscall\\n");
    return 0;
}
"""

# BPF 컴파일 및 실행
b = BPF(text=prog)
b.attach_kprobe(event="sys_clone", fn_name="trace_syscalls")

try:
    b.trace_print()
except KeyboardInterrupt:
    pass
```

---

**다음**: Cilium 실습 🔗
