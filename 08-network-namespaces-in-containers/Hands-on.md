# 08: Containers 실습

## 🎯 목표
- Docker 네트워킹 이해
- 컨테이너 격리 확인

## 🔬 실습 1: Docker 기본

```bash
# Docker 설치
sudo apt install docker.io

# 컨테이너 실행
docker run -d --name web nginx

# 네트워크 정보
docker inspect web | grep -i network

# 컨테이너의 namespace 확인
docker ps -q | head -1 | xargs -I {} \
  ls -la /proc/$(docker inspect -f '{{.State.Pid}}' {})/ns/
```

## 🔬 실습 2: Bridge 네트워크

```bash
# docker0 확인
ip link show docker0

# bridge 테이블
brctl show

# 컨테이너 간 통신
docker run -d --name web1 nginx
docker run -d --name web2 nginx
docker exec web1 ping -c 1 web2

# 또는 IP 직접 사용
docker exec web1 ping -c 1 172.17.0.3
```

## 🔬 실습 3: 포트 포워딩

```bash
# 포트 매핑 지정
docker run -d -p 8080:80 --name web nginx

# 테스트
curl http://localhost:8080

# iptables 규칙 확인
sudo iptables -t nat -L | grep DNAT
```

---

**다음**: Container Runtime 실습 🔗
