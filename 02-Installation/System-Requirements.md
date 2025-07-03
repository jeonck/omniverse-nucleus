# NVIDIA Omniverse Nucleus 시스템 요구사항

## 💻 하드웨어 및 소프트웨어 요구사항

NVIDIA Omniverse Nucleus 서버의 안정적인 운영을 위한 시스템 요구사항 및 권장 사양입니다.

## 📋 목차

1. [최소 시스템 요구사항](#최소-시스템-요구사항)
2. [권장 시스템 사양](#권장-시스템-사양)
3. [운영 체제 지원](#운영-체제-지원)
4. [네트워크 요구사항](#네트워크-요구사항)
5. [스토리지 요구사항](#스토리지-요구사항)
6. [데이터베이스 요구사항](#데이터베이스-요구사항)
7. [확장성 고려사항](#확장성-고려사항)
8. [성능 벤치마크](#성능-벤치마크)

---

## 🖥️ 최소 시스템 요구사항

### CPU 요구사항
- **프로세서**: Intel Core i5-8400 / AMD Ryzen 5 2600 이상
- **코어 수**: 6코어 이상
- **클럭 속도**: 2.8GHz 이상
- **아키텍처**: x86-64 (64-bit)
- **가상화 지원**: VT-x (Intel) / AMD-V (AMD) - 컨테이너 환경 시

### 메모리 요구사항
- **최소 RAM**: 16GB
- **권장 RAM**: 32GB 이상
- **메모리 타입**: DDR4-2400 이상
- **ECC 메모리**: 권장 (프로덕션 환경)

### 스토리지 요구사항
- **시스템 디스크**: 100GB 이상 (SSD 권장)
- **데이터 스토리지**: 500GB 이상 (프로젝트 규모에 따라)
- **임시 공간**: 50GB 이상
- **백업 공간**: 데이터 용량의 3배 이상

### GPU 요구사항 (선택사항)
- **NVIDIA GPU**: GTX 1060 / RTX 2060 이상
- **VRAM**: 6GB 이상
- **CUDA Compute**: 6.1 이상
- **드라이버**: 최신 Game Ready 또는 Studio 드라이버

---

## 🚀 권장 시스템 사양

### 소규모 팀 (5-15명)

```yaml
CPU:
  - 프로세서: Intel Core i7-10700K / AMD Ryzen 7 3700X
  - 코어 수: 8코어 16스레드
  - 클럭 속도: 3.6GHz 이상

메모리:
  - RAM: 32GB DDR4-3200
  - ECC: 권장

스토리지:
  - 시스템: 500GB NVMe SSD
  - 데이터: 2TB NVMe SSD (RAID 1)
  - 백업: 6TB HDD (RAID 5)

네트워크:
  - 인터페이스: 1Gbps Ethernet
  - 대역폭: 100Mbps 이상
```

### 중규모 팀 (15-50명)

```yaml
CPU:
  - 프로세서: Intel Xeon W-2245 / AMD Threadripper 3960X
  - 코어 수: 16코어 32스레드
  - 클럭 속도: 3.9GHz 이상

메모리:
  - RAM: 64GB DDR4-3200 ECC
  - 확장 슬롯: 128GB까지 확장 가능

스토리지:
  - 시스템: 1TB NVMe SSD
  - 데이터: 8TB NVMe SSD (RAID 10)
  - 캐시: 2TB NVMe SSD
  - 백업: 24TB HDD (RAID 6)

네트워크:
  - 인터페이스: 10Gbps Ethernet
  - 대역폭: 1Gbps 이상
  - 이중화: 권장
```

### 대규모 팀 (50명 이상)

```yaml
CPU:
  - 프로세서: Intel Xeon Gold 6248R / AMD EPYC 7543
  - 코어 수: 32코어 64스레드
  - 클럭 속도: 3.0GHz 이상
  - 듀얼 소켓: 권장

메모리:
  - RAM: 128GB DDR4-3200 ECC
  - 확장 슬롯: 512GB까지 확장 가능

스토리지:
  - 시스템: 2TB NVMe SSD
  - 데이터: 20TB NVMe SSD (RAID 10)
  - 캐시: 4TB NVMe SSD
  - 백업: 60TB SAS HDD (RAID 6)

네트워크:
  - 인터페이스: 25Gbps/40Gbps Ethernet
  - 대역폭: 10Gbps 이상
  - 이중화: 필수
```

---

## 🐧 운영 체제 지원

### Linux 지원 (권장)

```bash
# Ubuntu LTS 버전 (공식 검증된 OS)
Ubuntu 20.04 LTS (Focal Fossa)    ✅ 완전 지원
Ubuntu 22.04 LTS (Jammy Jellyfish) ✅ 완전 지원 (권장)
Ubuntu 24.04 LTS (Noble Numbat)    ⚠️ 공식 검증 대기중

# CentOS/RHEL
CentOS 8                           ✅ 지원
RHEL 8.x                          ✅ 지원
Rocky Linux 8.x                   ✅ 지원
AlmaLinux 8.x                     ✅ 지원

# SUSE
SLES 15 SP3+                      ✅ 지원
openSUSE Leap 15.3+               ✅ 지원

# Debian
Debian 11 (Bullseye)              ✅ 지원
Debian 12 (Bookworm)              ✅ 지원
```

### Windows 지원

```powershell
# Windows Server
Windows Server 2019               ✅ 지원
Windows Server 2022               ✅ 완전 지원

# Windows 10/11
Windows 10 Pro (1909+)            ⚠️ 개발/테스트용
Windows 11 Pro                    ⚠️ 개발/테스트용

# 필수 구성요소
.NET Framework 4.8+               ✅ 필수
Visual C++ Redistributable        ✅ 필수
PowerShell 5.1+                   ✅ 필수
```

### 컨테이너 지원

```yaml
Docker:
  - Docker Engine: 20.10.0 이상
  - Docker Compose: 2.0.0 이상
  - 플랫폼: linux/amd64, linux/arm64

Kubernetes:
  - 버전: 1.21 이상
  - 컨테이너 런타임: containerd, CRI-O
  - CNI: Calico, Flannel, Weave

Podman:
  - 버전: 3.0.0 이상
  - Rootless 모드 지원
```

---

## 🌐 네트워크 요구사항

### 대역폭 요구사항

```yaml
최소 요구사항:
  - 인터넷 연결: 100Mbps
  - 내부 네트워크: 1Gbps
  - 동시 사용자 1명당: 10Mbps

권장 요구사항:
  - 인터넷 연결: 1Gbps
  - 내부 네트워크: 10Gbps
  - 동시 사용자 1명당: 50Mbps

엔터프라이즈:
  - 인터넷 연결: 10Gbps
  - 내부 네트워크: 25Gbps+
  - 동시 사용자 1명당: 100Mbps
```

### 포트 요구사항 (2025 업데이트)

```bash
# 필수 포트
TCP 3009    # Nucleus 메인 서버 포트
TCP 8080    # 관리 인터페이스 포트
TCP 443     # HTTPS (프로덕션 환경)
TCP 80      # HTTP (리디렉션용)
UDP 60000   # Nucleus Bridge (Cloud 연동)

# 데이터베이스 포트
TCP 5432    # PostgreSQL
TCP 3306    # MySQL (선택사항)

# 캐시 및 메시징
TCP 6379    # Redis
TCP 5672    # RabbitMQ (선택사항)

# 모니터링 포트
TCP 9090    # Prometheus
TCP 3000    # Grafana
TCP 9093    # AlertManager

# 클러스터링 포트 (다중 노드 환경)
TCP 7946    # 클러스터 통신
UDP 7946    # 클러스터 발견
```

### 지연 시간 요구사항

```yaml
로컬 네트워크:
  - 최대 지연시간: 1ms
  - 패킷 손실: 0.01% 미만

광역 네트워크:
  - 최대 지연시간: 50ms
  - 패킷 손실: 0.1% 미만

인터넷 연결:
  - 최대 지연시간: 100ms
  - 패킷 손실: 0.5% 미만
```

---

## 💾 스토리지 요구사항

### 스토리지 타입별 성능

```yaml
시스템 디스크 (SSD 필수):
  - 순차 읽기: 3,500 MB/s 이상
  - 순차 쓰기: 3,000 MB/s 이상
  - 랜덤 읽기: 500,000 IOPS 이상
  - 랜덤 쓰기: 400,000 IOPS 이상

데이터 스토리지:
  - 순차 읽기: 2,000 MB/s 이상
  - 순차 쓰기: 1,800 MB/s 이상
  - 랜덤 읽기: 100,000 IOPS 이상
  - 랜덤 쓰기: 80,000 IOPS 이상

백업 스토리지:
  - 순차 읽기: 250 MB/s 이상
  - 순차 쓰기: 200 MB/s 이상
  - 용량: 데이터 볼륨의 3-5배
```

### 파일 시스템 권장사항

```bash
# Linux 파일 시스템
ext4      ✅ 권장 (범용성)
XFS       ✅ 권장 (대용량 파일)
ZFS       ✅ 권장 (고급 기능)
Btrfs     ⚠️ 안정성 검증 필요

# 마운트 옵션 권장사항
# ext4
/dev/sdb1 /var/lib/omni/nucleus/data ext4 defaults,noatime,errors=remount-ro 0 2

# XFS
/dev/sdb1 /var/lib/omni/nucleus/data xfs defaults,noatime,largeio 0 2

# ZFS
zpool create nucleus-data /dev/sdb
zfs set compression=lz4 nucleus-data
zfs set atime=off nucleus-data
```

### RAID 구성 권장사항

```yaml
시스템 디스크:
  - RAID 1 (미러링)
  - 2 x NVMe SSD

데이터 스토리지:
  - RAID 10 (소규모)
  - RAID 6 (대규모)
  - 4-8 x NVMe SSD

백업 스토리지:
  - RAID 5 (소규모)
  - RAID 6 (대규모)
  - 4-12 x SATA SSD/HDD
```

---

## 🗄️ 데이터베이스 요구사항

### PostgreSQL (권장)

```yaml
최소 버전: PostgreSQL 12
권장 버전: PostgreSQL 15+

메모리 설정:
  - shared_buffers: RAM의 25%
  - effective_cache_size: RAM의 75%
  - work_mem: 256MB
  - maintenance_work_mem: 2GB

연결 설정:
  - max_connections: 200
  - max_worker_processes: CPU 코어 수

성능 설정:
  - checkpoint_completion_target: 0.9
  - wal_buffers: 16MB
  - random_page_cost: 1.1 (SSD)
```

### 데이터베이스 크기 추정

```yaml
사용자당 데이터베이스 크기:
  - 기본 메타데이터: 10MB
  - 파일 인덱스: 1KB/파일
  - 사용자 세션: 5MB
  - 권한 정보: 1MB

프로젝트당 추정:
  - 소규모 (1,000파일): 50MB
  - 중규모 (10,000파일): 500MB
  - 대규모 (100,000파일): 5GB

총 데이터베이스 크기:
사용자 수 × 16MB + 프로젝트 크기 × 1.5배
```

---

## 📈 확장성 고려사항

### 수직 확장 (Scale-Up)

```yaml
CPU 확장:
  - 추가 CPU 소켓
  - 고클럭 프로세서 업그레이드
  - 더 많은 코어

메모리 확장:
  - 추가 메모리 모듈
  - 고용량 DIMM 교체
  - 최대 지원 용량까지

스토리지 확장:
  - 더 빠른 SSD 교체
  - 추가 스토리지 볼륨
  - 고성능 RAID 컨트롤러
```

### 수평 확장 (Scale-Out)

```yaml
로드 밸런싱:
  - 다중 Nucleus 인스턴스
  - 공유 스토리지 사용
  - 데이터베이스 클러스터링

지리적 분산:
  - 지역별 캐시 서버
  - CDN 활용
  - 지연 시간 최적화

마이크로서비스:
  - 기능별 서비스 분리
  - 컨테이너 오케스트레이션
  - 서비스 메시 활용
```

### 용량 계획

```bash
#!/bin/bash
# capacity_planning.sh

# 사용자 수별 리소스 계산
calculate_resources() {
    local users=$1
    local files_per_user=$2
    local avg_file_size_mb=$3
    
    # CPU 계산 (사용자 4명당 1코어)
    cpu_cores=$((users / 4 + 2))
    
    # 메모리 계산 (사용자당 1GB + 시스템 8GB)
    memory_gb=$((users + 8))
    
    # 스토리지 계산
    total_files=$((users * files_per_user))
    storage_gb=$((total_files * avg_file_size_mb / 1024))
    
    echo "=== $users 사용자 환경 리소스 계산 ==="
    echo "권장 CPU 코어: $cpu_cores"
    echo "권장 메모리: ${memory_gb}GB"
    echo "예상 스토리지: ${storage_gb}GB"
    echo "네트워크 대역폭: $((users * 50))Mbps"
}

# 사용 예제
calculate_resources 10 100 50   # 10명, 파일 100개/인, 50MB/파일
calculate_resources 50 500 100  # 50명, 파일 500개/인, 100MB/파일
calculate_resources 200 1000 200 # 200명, 파일 1000개/인, 200MB/파일
```

---

## 🔬 성능 벤치마크

### CPU 벤치마크

```bash
# CPU 성능 테스트
sysbench cpu --cpu-max-prime=20000 --threads=$(nproc) run

# 예상 결과
Intel i7-10700K:     ~15,000 events/sec
AMD Ryzen 7 3700X:   ~16,000 events/sec
Intel Xeon W-2245:   ~25,000 events/sec
AMD Threadripper:     ~35,000 events/sec
```

### 메모리 벤치마크

```bash
# 메모리 대역폭 테스트
sysbench memory --memory-total-size=10G run

# 예상 결과
DDR4-2400:           ~35 GB/s
DDR4-3200:           ~45 GB/s
DDR4-3600:           ~50 GB/s
```

### 스토리지 벤치마크

```bash
# 순차 읽기/쓰기 테스트
fio --name=seqread --rw=read --bs=1M --size=1G --numjobs=1
fio --name=seqwrite --rw=write --bs=1M --size=1G --numjobs=1

# 랜덤 읽기/쓰기 테스트
fio --name=randread --rw=randread --bs=4K --size=1G --numjobs=4
fio --name=randwrite --rw=randwrite --bs=4K --size=1G --numjobs=4

# SSD 예상 성능
SATA SSD:     550/520 MB/s (순차), 90K/80K IOPS (랜덤)
NVMe Gen3:    3,500/3,000 MB/s (순차), 500K/400K IOPS (랜덤)
NVMe Gen4:    7,000/6,500 MB/s (순차), 1M/800K IOPS (랜덤)
```

### 네트워크 벤치마크

```bash
# 네트워크 대역폭 테스트 (iperf3)
# 서버측
iperf3 -s

# 클라이언트측
iperf3 -c <server_ip> -t 60 -P 4

# 예상 결과
1Gbps Ethernet:      ~940 Mbps
10Gbps Ethernet:     ~9.4 Gbps
25Gbps Ethernet:     ~23 Gbps
```

---

## 🛡️ 보안 요구사항

### 하드웨어 보안

```yaml
TPM (Trusted Platform Module):
  - TPM 2.0 지원
  - 하드웨어 기반 암호화
  - 보안 부팅

하드웨어 암호화:
  - Self-Encrypting Drives (SED)
  - OPAL 2.0 지원
  - 256-bit AES 암호화
```

### 네트워크 보안

```yaml
방화벽:
  - 하드웨어 방화벽 권장
  - IDS/IPS 시스템
  - DDoS 보호

VPN:
  - IPsec VPN 지원
  - OpenVPN 호환성
  - WireGuard 지원
```

---

## 🧪 시스템 검증 도구

### 하드웨어 검증 스크립트

```bash
#!/bin/bash
# system_verification.sh

echo "=== NVIDIA Omniverse Nucleus 시스템 검증 ==="

# CPU 정보 확인
echo "1. CPU 정보"
lscpu | grep -E "Model name|CPU\(s\)|Thread|MHz"

# 메모리 정보 확인
echo "2. 메모리 정보"
free -h
echo "메모리 모듈 정보:"
dmidecode -t memory | grep -E "Size|Speed|Type" | head -10

# 스토리지 정보 확인
echo "3. 스토리지 정보"
lsblk -d -o NAME,SIZE,TYPE,MODEL
echo "디스크 성능 (간단 테스트):"
hdparm -t /dev/sda

# 네트워크 정보 확인
echo "4. 네트워크 정보"
ip link show
ethtool eth0 | grep Speed

# GPU 정보 확인 (NVIDIA)
echo "5. GPU 정보"
if command -v nvidia-smi >/dev/null; then
    nvidia-smi --query-gpu=name,memory.total,driver_version --format=csv
else
    echo "NVIDIA GPU 없음"
fi

# 운영체제 정보
echo "6. 운영체제 정보"
cat /etc/os-release | head -5

# 추천 설정 확인
echo "7. 추천 설정 확인"
CPU_CORES=$(nproc)
MEMORY_GB=$(free -g | grep Mem | awk '{print $2}')

if [ $CPU_CORES -ge 8 ]; then
    echo "✅ CPU 코어 수 충족: $CPU_CORES"
else
    echo "⚠️ CPU 코어 수 부족: $CPU_CORES (최소 8 권장)"
fi

if [ $MEMORY_GB -ge 32 ]; then
    echo "✅ 메모리 용량 충족: ${MEMORY_GB}GB"
else
    echo "⚠️ 메모리 용량 부족: ${MEMORY_GB}GB (최소 32GB 권장)"
fi

echo "시스템 검증 완료"
```

---

## 🔗 관련 문서

- [[01-Overview/Overview|개요]]
- [[02-Installation/Installation-Guide|설치 가이드]]
- [[03-Configuration/Configuration-Guide|구성 가이드]]
- [[04-API-Integration/API-Guide|API 통합]]
- [[05-Troubleshooting/Common-Issues|문제 해결]]
- [[06-Scripts-Tools/Utility-Scripts|스크립트 도구]]
- [[07-Kit-Client-SDK/Kit-Client-Python-API|Kit Client SDK]]

---

이 시스템 요구사항 가이드는 NVIDIA Omniverse Nucleus 서버의 안정적인 운영을 위한 모든 하드웨어 및 소프트웨어 요구사항을 포괄적으로 다룹니다. 실제 환경 구축 시 이 가이드를 참조하여 최적의 성능과 안정성을 확보할 수 있습니다.