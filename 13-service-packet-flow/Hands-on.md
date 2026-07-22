# 13: Service Packet Flow 실습

## 🎯 목표
- 패킷 흐름 추적
- tcpdump로 실시간 분석

## 🔬 실습 1: tcpdump로 모니터링

```bash
# Node에서 패킷 캡처
kubectl debug node/node-name -it --image=ubuntu

# 특정 Port의 패킷
sudo tcpdump -i any -n -nn port 80 | head -10

# Service IP로 필터링
sudo tcpdump -i any -n -nn dst <SERVICE_IP>

# pcap 파일로 저장
sudo tcpdump -i any -w capture.pcap port 80
```

## 🔬 실습 2: NAT 추적

```bash
# 클라이언트 Pod에서 요청
kubectl exec pod2 -- curl http://pod1-svc

# Node의 conntrack 확인
sudo conntrack -L | grep <SERVICE_IP>
```

## 🔬 실습 3: 완전한 흐름 추적

```bash
# 1. Pod 생성
kubectl run client --image=curlimages/curl -- sleep 3600
kubectl run server --image=nginx --port=80

# 2. Service 생성
kubectl expose pod server --port=80 --name=server-svc

# 3. 클라이언트에서 요청
kubectl exec client -- curl -v http://server-svc

# 4. Node에서 패킷 캡처
# kubectl debug node/... 후
# sudo tcpdump를 통해 모니터링
```

---

**다음**: NetworkPolicy 실습 🔗
