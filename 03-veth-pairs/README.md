# 03: veth pairs (Virtual Ethernet)

> **목표**: 격리된 Namespace 간에 가상 네트워크 연결 만들기
>
> **학습 시간**: 1.75시간
>
> **난이도**: ⭐⭐ 중급

## 🎯 핵심

- Namespace는 격리되어 있음 (이전 장)
- 하지만 통신도 필요함 → veth pair 사용
- veth = 두 개의 가상 이더넷 인터페이스가 연결된 쌍

## 📚 학습 순서

Theory.md → Architecture.md → Hands-on.md

## 실습 미리보기

```bash
# veth 쌍 생성
ip link add veth0 type veth peer name veth1

# 한쪽 끝을 Namespace로 이동
ip link set veth1 netns myspace
```

## ➡️ 다음: 04 Linux Bridge
