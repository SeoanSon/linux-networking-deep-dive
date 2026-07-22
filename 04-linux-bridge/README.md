# 04: Linux Bridge

> **목표**: 여러 개의 veth pair를 연결하는 L2 스위치 (Linux Bridge)
>
> **학습 시간**: 2.5시간
>
> **난이도**: ⭐⭐ 중급

## 🎯 핵심

- veth pair: 두 Namespace 연결
- Linux Bridge: 여러 개 Namespace를 한 번에 연결
- L2 스위칭, MAC 주소 학습, 브로드캐스트 처리

## 실습

```bash
brctl addbr br0
ip link set eth0 master br0
brctl show
```

## ➡️ 다음: 05 Routing
