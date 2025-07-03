# Nucleus Bridge 설정 가이드

## 🌉 개요

Nucleus Bridge는 Enterprise Nucleus Server를 Omniverse Cloud PaaS와 연결하여 웹 브라우저를 통한 RTX 스트리밍과 클라우드 서비스 접근을 가능하게 하는 새로운 기능입니다.

---

## 🎯 주요 기능

- **클라우드 PaaS 연동**: Omniverse Cloud와 안전한 연결
- **웹 RTX 스트리밍**: 브라우저에서 직접 Omniverse 접근  
- **이중 암호화**: TLS + 캡슐화로 강화된 보안
- **최소 성능 영향**: 기존 워크플로우 방해 없음

### 시스템 요구사항
```bash
- Linux Kernel 5.15+ (Ubuntu 22.04 기본)
- Enterprise Nucleus Server 2023.2.0+
- UDP 60000 아웃바운드 허용
- 공인 IP (NVIDIA 제공 필요)
```

---

## 🔧 설치 과정

### 1. Bridge 키 생성
```bash
#!/bin/bash
# setup_bridge.sh

echo "=== Nucleus Bridge 설정 ==="

# 서비스 중지
cd /opt/ove
docker compose down

# Bridge 키 생성
docker run --rm \
  -v $(pwd)/base_stack:/opt/ove/base_stack \
  nvcr.io/omniverse/nucleus-bridge-client-bootstrap:latest

# 공개키 확인
echo "생성된 공개키:"
cat base_stack/bridge/bridge.client.key.public

echo "⚠️ 이 공개키를 NVIDIA에 제공하세요."
```

### 2. 환경 설정
```bash
# nucleus-stack.env 편집
cat >> nucleus-stack.env << EOF

# ====== Nucleus Bridge 설정 ======
BRIDGE_ENABLED=1
BRIDGE_SERVER_HOST=bridge.omniverse.nvidia.com
BRIDGE_SERVER_PORT=60000
BRIDGE_CLIENT_ID=auto-generated
BRIDGE_RECONNECT_INTERVAL=30
BRIDGE_HEARTBEAT_INTERVAL=10
EOF
```

### 3. 서비스 재시작
```bash
# Bridge 설정으로 재시작
docker compose --env-file nucleus-stack.env up -d

# 상태 확인
sleep 60
docker logs nucleus-ingress-router --tail 20
```

---

## 🔍 상태 확인

### Bridge 연결 테스트
```bash
#!/bin/bash
# check_bridge.sh

echo "=== Bridge 상태 확인 ==="

# 컨테이너 확인
CONTAINER=$(docker ps --filter "name=nucleus-ingress-router" --format "{{.ID}}")
if [ -n "$CONTAINER" ]; then
    echo "✅ Bridge 컨테이너 실행 중"
else
    echo "❌ Bridge 컨테이너 없음"
    exit 1
fi

# 연결 상태
docker exec nucleus-ingress-router \
    netstat -an | grep :60000 && echo "✅ UDP 연결 활성" || echo "❌ 연결 없음"

# 설정 파일
[ -f "/opt/ove/base_stack/bridge/bridge.map" ] && echo "✅ bridge.map 존재" || echo "❌ bridge.map 없음"
[ -f "/opt/ove/base_stack/bridge/bridge.client.key" ] && echo "✅ 클라이언트 키 존재" || echo "❌ 키 없음"
```

---

## 🌐 클라우드 스트리밍

Bridge 활성화 후 NVIDIA에서 제공하는 URL로 접근:
```
https://your-company.omniverse.nvidia.com
```

### 스트리밍 최적화
```yaml
# nucleus-stack.env 추가 설정
BRIDGE_STREAMING_QUALITY=high
BRIDGE_MAX_BITRATE=50000
BRIDGE_TARGET_FPS=60
BRIDGE_LOW_LATENCY=true
BRIDGE_ADAPTIVE_BITRATE=true
```

---

## 🔧 문제 해결

### 일반적인 문제

#### 1. UDP 포트 차단
```bash
sudo ufw allow out 60000/udp
sudo systemctl restart ufw
```

#### 2. 키 재생성
```bash
rm -f /opt/ove/base_stack/bridge/bridge.client.key*
docker run --rm \
  -v /opt/ove/base_stack:/opt/ove/base_stack \
  nvcr.io/omniverse/nucleus-bridge-client-bootstrap:latest
```

#### 3. 스트리밍 품질 조정
```yaml
# 저대역폭 환경용
BRIDGE_TARGET_FPS=30
BRIDGE_MAX_BITRATE=15000
BRIDGE_MIN_BITRATE=5000
```

---

## ✅ 체크리스트

### 사전 준비
- [ ] Linux Kernel 5.15+ 확인
- [ ] Enterprise Nucleus Server 2023.2.0+
- [ ] 공인 IP 주소 확인
- [ ] UDP 60000 아웃바운드 허용
- [ ] NVIDIA 지원팀 연락 및 승인

### Bridge 설정
- [ ] 암호화 키 생성
- [ ] 공개키 NVIDIA 전송
- [ ] nucleus-stack.env 설정
- [ ] 서비스 재시작

### 연결 확인
- [ ] Bridge 컨테이너 정상 실행
- [ ] UDP 연결 테스트 통과
- [ ] 클라우드 스트리밍 접근 가능
- [ ] 사용자 권한 설정 완료

---

**중요**: Nucleus Bridge는 NVIDIA의 사전 승인이 필요한 기능입니다. 설정 전에 반드시 enterprise-support@nvidia.com으로 연락하여 승인을 받으시기 바랍니다.