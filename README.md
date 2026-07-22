# 📚 Linux 네트워킹 심화 학습: Kubernetes, CNI, eBPF, Cilium까지

> **목표**: 제로부터 시작해서 Linux 커널의 네트워킹 스택을 완전히 이해하고, 최종적으로 Cilium 아키텍처를 설명할 수 있는 Kubernetes 네트워킹 전문가 되기
>
> **대상**: Azure Solutions Engineer 및 DevOps/Platform Engineer
>
> **참고**: 이것은 인증 시험 대비가 아닌 **기술 깊이**를 위한 학습 자료입니다.

---

## 🎯 학습 철학

### 왜 이 과정이 필요한가?

```
문제: "Kubernetes 네트워킹이 복잡하다"
원인: 추상화 계층을 거쳐서 학습했기 때문
해결: Linux 커널부터 시작해서 위로 올라가기
```

이 저장소는 **역순**으로 배웠던 모든 추상화를 **원래대로 벗겨내는** 학습 여정입니다.

### 핵심 원칙

- ✅ **WHY > HOW** - 항상 '왜'부터 시작
- ✅ **First Principles** - 기초부터 차근차근 (커널 지식 X 가정)
- ✅ **Layered Learning** - 각 챕터가 이전 위에 구축됨
- ✅ **Hands-on First** - 실제 명령어로 배우기 (스크린샷 X)
- ✅ **Azure Focused** - 모든 개념을 AKS와 연결
- ✅ **Enterprise Ready** - 실제 고객 시나리오와 설계 트레이드오프

---

## 📖 전체 학습 로드맵

```
Part 1: Linux 기초
├─ 01: Linux Process & IPC
├─ 02: Network Namespace (3가지 관점)
├─ 03: veth pair (가상 이더넷)
└─ 04: Linux Bridge (L2 스위칭)

Part 2: Linux 네트워킹
├─ 05: Routing (커널 라우팅 테이블)
├─ 06: iptables (패킷 필터링/NAT)
└─ 07: netfilter (iptables의 실제 구현)

Part 3: 컨테이너 네트워킹
├─ 08: Network Namespace in Containers
├─ 09: Container Runtime Networking (Docker/containerd)
└─ 10: Kubernetes 네트워킹 (Pod IP, Service)

Part 4: Kubernetes 심화
├─ 11: CNI (Container Network Interface)
├─ 12: kube-proxy (Service 구현)
├─ 13: Service Packet Flow (한 패킷의 여행)
└─ 14: NetworkPolicy (L3/L4 정책)

Part 5: eBPF & Cilium
├─ 15: eBPF (확장 BPF, 커널 프로그래밍)
├─ 16: Cilium (eBPF 기반 네트워킹)
├─ 17: Cilium NetworkPolicy (정책 구현)
└─ 18: 관찰성 (추적, 메트릭, 로그)

Part 6: 실전 AKS
└─ 19: 실제 AKS 네트워킹 구축 및 문제 해결
```

---

## 🏗️ 각 챕터 구조

모든 챕터는 **7개의 문서**로 구성됩니다:

### 📋 README.md
```
개요 및 학습 목표
챕터 개요
선수 지식
학습 시간 추정
```

### 🧠 Theory.md
```
개념 설명 (왜 이게 필요한가?)
역사적 배경 (언제 생겼는가?)
용어 어원 (말의 의미)
아키텍처 (어떻게 작동하는가?)
AKS에서의 모습 (클라우드 관점)
```

### 💻 Hands-on.md
```
0️⃣ 사전 준비 (환경 설정)
1️⃣ 기본 명령어 (직접 해보기)
2️⃣ 심화 실습 (이해 검증)
3️⃣ 트러블슈팅 (뭔가 잘못됐을 때)
모든 명령어는 copy-paste ready
```

### 🔍 Architecture.md
```
시스템 아키텍처 다이어그램 (ASCII 포함)
컴포넌트 간 관계
데이터 플로우
커널 관점 설명
```

### 📡 PacketFlow.md
```
한 패킷의 여행
커널 스택 통과
각 단계별 상태 변화
ASCII 패킷 흐름도
```

### 🔧 Troubleshooting.md
```
흔한 문제와 해결책
디버깅 명령어
성능 문제 진단
로그 해석
```

### 💼 Interview.md
```
엔터프라이즈 관점의 면접 질문
답변 및 심화 질문
실제 고객 시나리오
설계 트레이드오프
```

---

## 🚀 빠른 시작

### 사전 요구사항

```bash
# Linux 환경 (Ubuntu 22.04 권장)
# - WSL2, Docker Desktop, 또는 Linux VM
# - root 또는 sudo 권한

# 설치 필요한 도구들
sudo apt-get update
sudo apt-get install -y \
  net-tools \
  iproute2 \
  bridge-utils \
  iptables \
  tcpdump \
  curl \
  jq \
  docker.io \
  golang-1.21
```

### 학습 방법

```
1️⃣ README.md 읽기 (개요 파악)
   ↓
2️⃣ Theory.md 정독 (개념 이해)
   ↓
3️⃣ Architecture.md 분석 (구조 파악)
   ↓
4️⃣ Hands-on.md 실습 (직접 실행)
   ↓
5️⃣ PacketFlow.md 추적 (흐름 이해)
   ↓
6️⃣ Troubleshooting.md 참고 (문제 해결)
   ↓
7️⃣ Interview.md 검증 (개념 확인)
```

---

## 📊 챕터별 학습 시간

| 챕터 | 주제 | 이론 | 실습 | 합계 |
|------|------|------|------|------|
| 01 | Linux Process | 30분 | 1시간 | 1.5시간 |
| 02 | Network Namespace | 1시간 | 1.5시간 | 2.5시간 |
| 03 | veth pairs | 45분 | 1시간 | 1.75시간 |
| 04 | Linux Bridge | 1시간 | 1.5시간 | 2.5시간 |
| 05 | Routing | 1시간 | 1.5시간 | 2.5시간 |
| 06-07 | iptables/netfilter | 1.5시간 | 2시간 | 3.5시간 |
| 08-09 | Container Networking | 1시간 | 1.5시간 | 2.5시간 |
| 10 | K8s Networking | 1.5시간 | 2시간 | 3.5시간 |
| 11-14 | CNI, kube-proxy, Service | 2시간 | 2.5시간 | 4.5시간 |
| 15 | eBPF | 1.5시간 | 2시간 | 3.5시간 |
| 16-17 | Cilium | 1.5시간 | 2시간 | 3.5시간 |
| 18 | 관찰성 | 1시간 | 1.5시간 | 2.5시간 |
| 19 | 실전 AKS | 2시간 | 3시간 | 5시간 |
| **합계** | | **18시간** | **24시간** | **42시간** |

---

## 🎓 학습 로그 추적

각 챕터를 완료하면 다음을 체크리스트로 활용하세요:

```markdown
# 01-linux-processes 학습 완료

- [ ] README.md 읽음 (개요 이해)
- [ ] Theory.md 정독 (개념 학습)
- [ ] Hands-on.md 실습 완료 (직접 실행)
- [ ] Architecture.md 분석 (시스템 이해)
- [ ] PacketFlow.md 검토 (흐름 파악)
- [ ] Troubleshooting.md 사례 확인
- [ ] Interview.md 질문 답변 가능
- [ ] 다음 챕터 선수 지식 확인 ✓
```

---

## 🔗 각 챕터 선수 지식

```
01 Linux Process
  └─→ 02 Network Namespace (반드시 필요)
       └─→ 03 veth pairs (필수)
            └─→ 04 Linux Bridge (필수)
                 └─→ 05 Routing (필수)
                      └─→ 06 iptables
                           └─→ 07 netfilter
                                └─→ 08 Containers
                                     └─→ 09 Container Runtime
                                          └─→ 10 Kubernetes
                                               └─→ 11 CNI
                                                    └─→ 12 kube-proxy
                                                         └─→ 13 Service Flow
                                                              └─→ 14 NetworkPolicy
                                                                   └─→ 15 eBPF
                                                                        └─→ 16 Cilium
                                                                             └─→ 17 Cilium Policy
                                                                                  └─→ 18 Observability
                                                                                       └─→ 19 Real-world AKS
```

---

## 🏢 Azure Solutions Engineer 관점

### 각 챕터의 고객 관련성

| 챕터 | 고객 시나리오 | 설계 트레이드오프 |
|------|-------------|-----------------|
| 01-04 | 컨테이너 기본 | 성능 vs 격리 |
| 05-07 | 트래픽 라우팅/필터링 | 유연성 vs 복잡도 |
| 08-10 | Kubernetes 기초 | 추상화 vs 보이성 |
| 11-14 | Pod 통신, Service | CNI 선택 |
| 15-17 | 고성능 정책 | eBPF 복잡도 vs 성능 |
| 18-19 | 운영/모니터링 | 가시성 vs 오버헤드 |

---

## 💡 핵심 원칙: "한 패킷의 여행"

이 저장소 전체는 **한 패킷이 Pod A에서 Pod B로 가는 과정**을 점진적으로 이해하는 것입니다.

```
Part 1 (01-04): 패킷이 지나는 물리적 경로 이해
  ↓
Part 2 (05-07): 패킷 필터링 및 라우팅 규칙
  ↓
Part 3 (08-10): 컨테이너/Kubernetes 추상화
  ↓
Part 4 (11-14): Service를 통한 패킷 변환
  ↓
Part 5 (15-17): eBPF로 커널에서의 최적화
  ↓
Part 6 (18-19): 실전에서의 디버깅/모니터링
```

---

## 📚 참고 자료

### 필수 읽을 것
- Linux Kernel Documentation: [Networking](https://www.kernel.org/doc/html/latest/networking/)
- eBPF 공식 문서: [ebpf.io](https://ebpf.io)
- Cilium 아키텍처: [Cilium Architecture](https://docs.cilium.io/en/latest/concepts/architecture/)

### 추천 도서
- "Linux Kernel Development" by Robert Love
- "TCP/IP Illustrated" (Volume 1 & 2)
- "The eBPF Programming Journey"

### 온라인 커뮤니티
- Cilium Slack
- CNCF Networking Slack
- Kubernetes 한글 커뮤니티

---

## 📝 학습 체크리스트

완료 기준:
- [ ] 각 챕터의 7개 문서를 모두 읽음
- [ ] Hands-on의 모든 실습을 직접 실행함
- [ ] PacketFlow 다이어그램을 손으로 그릴 수 있음
- [ ] Interview 질문에 대답할 수 있음
- [ ] 고객에게 설명할 수 있음

---

## 🤝 기여 및 피드백

이 자료는 **커뮤니티 주도 프로젝트**입니다.

- 오류 발견 시 Issue 제출
- 개선 사항은 Pull Request
- 경험담/팁 공유 Welcome

---

## 📄 라이선스

MIT License - 자유롭게 사용, 수정, 배포 가능

---

**시작하기**: `cd 01-linux-processes && cat README.md`

**행운을 빕니다! 🚀**
