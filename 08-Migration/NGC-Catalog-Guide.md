# NGC Catalog 사용 가이드

## 🌟 개요

NGC (NVIDIA GPU Cloud) Catalog는 Omniverse Launcher를 대체하는 새로운 배포 플랫폼입니다. 엔터프라이즈급 보안, 자동화, 팀 협업 기능을 제공합니다.

---

## 🎯 주요 특징

### NGC Catalog vs Launcher

```yaml
NGC Catalog 장점:
  - 엔터프라이즈급 보안 (API 키, RBAC)
  - 자동화 지원 (CLI, API)
  - 글로벌 CDN (빠른 다운로드)
  - 팀 협업 기능
  - 체계적인 버전 관리

Launcher 대비 개선:
  - 더 안정적인 배포
  - 향상된 성능
  - 클라우드 네이티브 설계
  - 사용량 분석 대시보드
```

---

## 🔐 계정 설정

### 1. NGC 계정 생성

1. **웹사이트**: https://ngc.nvidia.com
2. **NVIDIA Developer 계정으로 로그인**  
3. **프로필 완성** (회사 정보 필수)
4. **이용약관 동의**

### 2. API Key 생성

```bash
# NGC 웹에서:
# 프로필 → Setup → Generate API Key
# 이름: "Omniverse-Production"

# 환경 변수 설정
export NGC_API_KEY="your_api_key_here"
```

### 3. NGC CLI 설치

```bash
# 다운로드 및 설치
wget -O ngccli_linux.zip https://ngc.nvidia.com/downloads/ngccli_linux.zip
unzip ngccli_linux.zip
chmod +x ngc-cli/ngc
export PATH="$PATH:$(pwd)/ngc-cli"

# 설정
ngc config set
```

---

## 📥 Nucleus 다운로드

### 기본 다운로드

```bash
# 최신 버전
ngc registry resource download-version \
  nvidia/omniverse/nucleus-compose-stack:latest

# 특정 버전  
ngc registry resource download-version \
  nvidia/omniverse/nucleus-compose-stack:2024.1.0
```

### 자동화 배포 스크립트

```bash
#!/bin/bash
# auto_deploy_nucleus.sh

set -e

VERSION="latest"
DEPLOY_DIR="/opt/omniverse"

echo "🚀 NGC에서 Nucleus 자동 배포"

# NGC 인증 확인
if ! ngc config current &>/dev/null; then
    echo "❌ NGC 설정 필요: ngc config set"
    exit 1
fi

# 백업 및 다운로드
if [ -d "$DEPLOY_DIR" ]; then
    sudo mv "$DEPLOY_DIR" "${DEPLOY_DIR}_backup_$(date +%Y%m%d)"
fi

sudo mkdir -p "$DEPLOY_DIR"
cd "$DEPLOY_DIR"

ngc registry resource download-version \
  "nvidia/omniverse/nucleus-compose-stack:$VERSION" \
  --dest .

# 설정 및 배포
sudo chown -R $USER:$USER .
cp nucleus-stack.env.template nucleus-stack.env

# 기본 설정
sed -i "s/^WEB_PORT=.*/WEB_PORT=8080/" nucleus-stack.env
DB_PASS=$(openssl rand -base64 32)
sed -i "s/^DB_PASSWORD=.*/DB_PASSWORD=$DB_PASS/" nucleus-stack.env

# 시작
docker compose --env-file nucleus-stack.env up -d

# 검증
sleep 30
if curl -s http://localhost:8080/health > /dev/null; then
    echo "✅ 배포 성공: http://localhost:8080"
else
    echo "❌ 배포 실패"
    exit 1
fi
```

---

## 🔌 API 활용

### Python 클라이언트

```python
# ngc_client.py
import requests
from typing import Dict, List

class NGCClient:
    def __init__(self, api_key: str):
        self.api_key = api_key
        self.base_url = "https://api.ngc.nvidia.com/v2"
        self.headers = {
            "Authorization": f"ApiKey {api_key}",
            "Content-Type": "application/json"
        }
    
    def search_resources(self, query: str) -> Dict:
        """리소스 검색"""
        response = requests.get(
            f"{self.base_url}/search/catalog/resources",
            headers=self.headers,
            params={"q": query, "org": "nvidia", "type": "container"}
        )
        return response.json()
    
    def list_versions(self, resource: str) -> Dict:
        """버전 목록"""
        response = requests.get(
            f"{self.base_url}/org/nvidia/team/omniverse/resources/{resource}/versions",
            headers=self.headers
        )
        return response.json()
    
    def get_latest_version(self, resource: str) -> str:
        """최신 버전 조회"""
        versions = self.list_versions(resource)
        return versions["versions"][0]["name"]

# 사용 예제
client = NGCClient("your_api_key")
latest = client.get_latest_version("nucleus-compose-stack")
print(f"최신 버전: {latest}")
```

### 자동 업데이트 시스템

```python
# auto_updater.py
import schedule
import time
import subprocess
from ngc_client import NGCClient

class NucleusUpdater:
    def __init__(self, api_key: str):
        self.client = NGCClient(api_key)
        self.current_version = self.get_current_version()
    
    def get_current_version(self) -> str:
        """현재 버전 확인"""
        try:
            result = subprocess.run(
                ["docker", "inspect", "nucleus", "--format", "{{.Config.Labels.version}}"],
                capture_output=True, text=True
            )
            return result.stdout.strip()
        except:
            return "unknown"
    
    def check_updates(self):
        """업데이트 확인"""
        latest = self.client.get_latest_version("nucleus-compose-stack")
        
        if self.current_version != latest:
            print(f"🚀 업데이트 발견: {self.current_version} → {latest}")
            
            if self.update_nucleus(latest):
                print("✅ 업데이트 완료")
                self.current_version = latest
            else:
                print("❌ 업데이트 실패")
    
    def update_nucleus(self, version: str) -> bool:
        """Nucleus 업데이트"""
        try:
            # 백업
            subprocess.run(["docker", "exec", "nucleus_db", 
                          "pg_dump", "-U", "nucleus", "nucleus"], 
                         stdout=open(f"/backup/db_{int(time.time())}.sql", "w"))
            
            # 다운로드
            subprocess.run([
                "ngc", "registry", "resource", "download-version",
                f"nvidia/omniverse/nucleus-compose-stack:{version}",
                "--dest", "/tmp/nucleus-update"
            ])
            
            # 재배포
            subprocess.run(["docker", "compose", "down"])
            subprocess.run(["cp", "-r", "/tmp/nucleus-update/*", "/opt/omniverse/"])
            subprocess.run(["docker", "compose", "up", "-d"])
            
            # 검증
            time.sleep(60)
            return subprocess.run(["curl", "-f", "http://localhost:8080/health"]).returncode == 0
            
        except Exception as e:
            print(f"오류: {e}")
            return False

# 스케줄링
updater = NucleusUpdater("your_api_key")
schedule.every().day.at("02:00").do(updater.check_updates)

while True:
    schedule.run_pending()
    time.sleep(3600)
```

---

## 🛡️ 보안 Best Practices

### API 키 관리

```bash
# 환경 변수
export NGC_API_KEY="your_key"

# Docker Secrets
echo "$NGC_API_KEY" | docker secret create ngc_api_key -

# Kubernetes Secrets  
kubectl create secret generic ngc-key \
  --from-literal=api-key="$NGC_API_KEY"
```

### 접근 제어

```python
# auth_validator.py
import requests

def validate_ngc_key(api_key: str) -> bool:
    """NGC API 키 검증"""
    try:
        response = requests.get(
            'https://api.ngc.nvidia.com/v2/org',
            headers={'Authorization': f'ApiKey {api_key}'}
        )
        return response.status_code == 200
    except:
        return False
```

---

## 🚀 CI/CD 통합

### GitHub Actions

```yaml
# .github/workflows/nucleus-deploy.yml
name: Auto Deploy Nucleus

on:
  schedule:
    - cron: '0 2 * * 1'  # 매주 월요일 2시
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
    - name: Setup NGC CLI
      run: |
        wget -O ngccli_linux.zip https://ngc.nvidia.com/downloads/ngccli_linux.zip
        unzip ngccli_linux.zip
        chmod +x ngc-cli/ngc
        echo "$(pwd)/ngc-cli" >> $GITHUB_PATH
    
    - name: Check for updates
      id: check
      run: |
        ngc config set --api-key ${{ secrets.NGC_API_KEY }} --org nvidia
        LATEST=$(ngc registry resource list-versions nvidia/omniverse/nucleus-compose-stack --format json | jq -r '.versions[0].name')
        echo "version=$LATEST" >> $GITHUB_OUTPUT
    
    - name: Deploy if needed
      run: |
        # 배포 로직
        ./deploy.sh ${{ steps.check.outputs.version }}
```

---

## 📊 모니터링

### 사용량 추적

```bash
# 다운로드 이력
ngc registry resource list-downloads \
  --org nvidia --team omniverse --limit 10

# 팀별 사용량
ngc team usage summary --period last-30-days

# 인기 리소스
ngc registry analytics popular --period last-week
```

### 성능 메트릭

```python
# performance_tracker.py
import time
import requests

def measure_download_speed(url: str) -> float:
    """다운로드 속도 측정"""
    start = time.time()
    response = requests.get(url, stream=True)
    
    size = int(response.headers.get('content-length', 0))
    for chunk in response.iter_content(8192):
        pass
    
    duration = time.time() - start
    return (size / 1024 / 1024) / duration  # MB/s

speed = measure_download_speed("your_resource_url")
print(f"다운로드 속도: {speed:.2f} MB/s")
```

---

## ✅ 체크리스트

### 초기 설정
- [ ] NGC 계정 생성
- [ ] API 키 발급
- [ ] NGC CLI 설치
- [ ] 팀/조직 구성

### 배포 준비  
- [ ] Nucleus 다운로드
- [ ] 환경 설정
- [ ] Docker 배포
- [ ] 기능 테스트

### 운영 환경
- [ ] SSL 설정
- [ ] 백업 구성
- [ ] 모니터링 설정
- [ ] 자동 업데이트
- [ ] 보안 검토

---

이 가이드를 따라 NGC Catalog를 효과적으로 활용하여 안정적이고 확장 가능한 Omniverse 환경을 구축할 수 있습니다.