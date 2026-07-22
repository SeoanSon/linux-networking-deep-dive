# 02: Linux Network Namespace

> **목표**: Linux에서 네트워크를 격리하는 방법 (Network Namespace)을 배우기
>
> **학습 시간**: 2.5시간 (이론 1시간 + 실습 1.5시간)
>
> **난이도**: ⭐⭐ 중급

---

## 📍 위치

```
01: Linux Process & IPC
  ↓
02: Linux Network Namespace ← YOU ARE HERE
  ↓
03: veth pairs
```

---

## 🎯 학습 목표

- ✅ Network Namespace 개념
- ✅ Namespace 생성 및 관리
- ✅ 격리된 네트워크 환경 이해
- ✅ AKS Pod 격리와의 연결

---

## 📚 이 장의 구성

1. **Theory.md** - Network Namespace의 기초
2. **Architecture.md** - 격리 메커니즘
3. **Hands-on.md** - 직접 Namespace 만들기
4. **PacketFlow.md** - 패킷이 Namespace를 어떻게 통과하는가
5. **Troubleshooting.md** - 문제 해결
6. **Interview.md** - 면접 대비

---

## ❓ 핵심 질문

1. **Namespace란?** - 커널 기능으로 시스템의 일부를 격리
2. **Network Namespace는?** - 네트워크 인터페이스와 라우팅 테이블을 격리
3. **왜 필요한가?** - 컨테이너/Pod를 독립적인 네트워크 환경으로 만들기 위해
4. **Kubernetes와의 관계?** - 각 Pod이 자신의 Network Namespace를 가짐

---

## 🏗️ 핵심 내용 미리보기

```
호스트 Network Namespace (global)
├─ eth0 인터페이스
├─ 라우팅 테이블
└─ ARP 테이블

새로운 Network Namespace (격리됨)
├─ lo (localhost만)
├─ 자체 라우팅 테이블
└─ 외부 네트워크 접근 불가 (기본)
```

---

## 🚀 실습 미리보기

```bash
# Namespace 생성
ip netns add myspace

# Namespace 내에서 명령어 실행
ip netns exec myspace ip link show

# 현재 Namespace의 인터페이스
ip link show
```

---

## 🏢 AKS 관점

**Pod = Network Namespace의 묶음**

```
Pod A
└─ Network Namespace A
   ├─ eth0 (10.244.1.5)
   ├─ lo (127.0.0.1)
   └─ 자체 라우팅 테이블

호스트 커널
└─ veth 쌍으로 연결
   └─ Pod 네트워크 인터페이스와 호스트 연결
```

---

## 🎓 완료 기준

- [ ] Namespace 개념을 설명할 수 있다
- [ ] 새로운 Namespace를 생성할 수 있다
- [ ] `ip netns` 명령어를 사용할 수 있다
- [ ] Namespace 간의 격리를 이해한다
- [ ] Pod의 네트워크 격리 이유를 설명할 수 있다

---

## ➡️ 다음으로

**03: veth pairs** - 격리된 Namespace들을 연결하는 가상 네트워크 기기
