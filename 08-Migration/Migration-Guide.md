# Omniverse Launcher 마이그레이션 가이드

## 🚨 중요 공지

**NVIDIA Omniverse Launcher는 2025년 10월 1일부로 지원이 중단됩니다.**

이 가이드는 기존 Launcher 기반 환경에서 새로운 NGC Catalog 기반 환경으로 안전하게 마이그레이션하는 방법을 제공합니다.

---

## 🔄 마이그레이션 개요

### 변경사항 요약

- **지원 중단**: Omniverse Launcher, Nucleus Workstation
- **대체 방안**: Enterprise Nucleus Server (NGC Catalog)
- **새로운 접근**: 직접 다운로드, 클라우드 네이티브 배포
- **향상된 기능**: Nucleus Bridge, 개선된 성능

### 마이그레이션 혜택

- 더 안정적인 엔터프라이즈급 기능
- 클라우드 PaaS 연동 가능
- 개선된 보안 및 성능
- 더 나은 확장성

---

## 🛠️ 단계별 마이그레이션

### 1단계: 환경 준비

```bash
# 현재 환경 백업
mkdir -p ~/omniverse_backup_$(date +%Y%m%d)
cp -r ~/.nvidia-omniverse ~/omniverse_backup_$(date +%Y%m%d)/

# NGC CLI 설치
wget https://ngc.nvidia.com/downloads/ngccli_linux.zip
unzip ngccli_linux.zip && chmod +x ngc-cli/ngc
export PATH="$PATH:$(pwd)/ngc-cli"
```

### 2단계: Enterprise Nucleus Server 배포

```bash
# NGC에서 Nucleus 다운로드
ngc registry resource download-version \
  nvidia/omniverse/nucleus-compose-stack:latest

# Docker Compose 배포
cd nucleus-compose-stack
cp nucleus-stack.env.template nucleus-stack.env

# 환경 설정
cat >> nucleus-stack.env << EOF
WEB_PORT=8080
DATA_ROOT=/opt/ove/data
DB_PASSWORD=$(openssl rand -base64 32)
EOF

# 배포 실행
docker compose up -d
```

### 3단계: 데이터 마이그레이션

```python
import omni.client
import os

def migrate_projects():
    """기존 프로젝트를 새 Nucleus로 마이그레이션"""
    
    source_dir = os.path.expanduser("~/.nvidia-omniverse/data")
    target_url = "omniverse://localhost/Projects"
    
    for project in os.listdir(source_dir):
        project_path = os.path.join(source_dir, project)
        if os.path.isdir(project_path):
            print(f"마이그레이션 중: {project}")
            
            # 프로젝트 폴더 생성
            omni.client.create_folder(f"{target_url}/{project}")
            
            # 파일 업로드
            for root, dirs, files in os.walk(project_path):
                for file in files:
                    local_path = os.path.join(root, file)
                    relative_path = os.path.relpath(local_path, project_path)
                    remote_path = f"{target_url}/{project}/{relative_path}"
                    
                    with open(local_path, 'rb') as f:
                        omni.client.write_file(remote_path, f.read())
            
            print(f"✅ {project} 마이그레이션 완료")

if __name__ == "__main__":
    migrate_projects()
```

---

## 🌟 새로운 기능 활용

### Nucleus Bridge 설정 (클라우드 연동)

```bash
# Nucleus Bridge 활성화
echo "BRIDGE_ENABLED=1" >> nucleus-stack.env

# 브리지 키 생성
docker run --rm -v $(pwd)/base_stack:/opt/ove/base_stack \
  nvcr.io/omniverse/nucleus-bridge-client-bootstrap:latest

# 공개키 확인 (NVIDIA에 제공 필요)
cat base_stack/bridge/bridge.client.key.public
```

### Navigator 웹 인터페이스

새로운 독립 실행형 Navigator는 웹 브라우저에서 직접 실행됩니다:

- **URL**: http://localhost:8080
- **기능**: 파일 관리, 사용자 권한, 협업 도구
- **개선사항**: 향상된 UI, 더 빠른 성능

---

## 🔧 문제 해결

### 일반적인 문제들

**Q: 데이터 마이그레이션이 느림**
```bash
# 청크 업로드 활성화
echo "NUCLEUS_UPLOAD_CHUNK_SIZE=64" >> nucleus-stack.env
docker compose restart
```

**Q: SSL 인증서 설정**
```bash
# Let's Encrypt 사용
certbot certonly --standalone -d your-nucleus.domain.com
echo "SSL_ENABLED=1" >> nucleus-stack.env
echo "SSL_CERT_PATH=/etc/letsencrypt/live/your-nucleus.domain.com/fullchain.pem" >> nucleus-stack.env
```

**Q: 성능 최적화**
```bash
# PostgreSQL 튜닝
docker exec -it nucleus_db psql -U nucleus -c "
ALTER SYSTEM SET shared_buffers = '2GB';
ALTER SYSTEM SET effective_cache_size = '6GB';
SELECT pg_reload_conf();
"
```

---

## 📚 추가 리소스

### 공식 문서
- [NGC Catalog](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/omniverse)
- [Enterprise Nucleus 가이드](https://docs.omniverse.nvidia.com/nucleus/latest/enterprise/)
- [마이그레이션 지원](https://developer.nvidia.com/omniverse/legacy-tools)

### 커뮤니티 지원
- [NVIDIA Developer Forums](https://forums.developer.nvidia.com/c/omniverse/)
- [Discord](https://discord.gg/nvidiaomniverse)

---

## ✅ 마이그레이션 체크리스트

### 마이그레이션 전
- [ ] 현재 환경 백업 완료
- [ ] NGC 계정 생성 및 API 키 발급
- [ ] 시스템 요구사항 확인
- [ ] 네트워크 포트 설정 확인

### 마이그레이션 중
- [ ] Enterprise Nucleus Server 배포
- [ ] 데이터베이스 마이그레이션
- [ ] 프로젝트 파일 전송
- [ ] 사용자 계정 재생성
- [ ] 권한 설정 복구

### 마이그레이션 후
- [ ] 기능 테스트 완료
- [ ] 성능 벤치마크 확인
- [ ] 사용자 교육 실시
- [ ] 모니터링 설정
- [ ] 백업 자동화 구성

---

**중요**: 프로덕션 환경 마이그레이션 전에 반드시 테스트 환경에서 전체 과정을 검증하시기 바랍니다.

**지원**: 마이그레이션 과정에서 문제가 발생하면 NVIDIA 지원팀에 문의하거나 커뮤니티 포럼을 활용하세요.