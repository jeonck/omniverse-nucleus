# Nucleus 설치 및 배포 가이드

## 🖥️ 시스템 요구사항

### 최소 사양
- **OS**: Ubuntu 20.04 LTS, CentOS 8, Windows Server 2019
- **CPU**: 8 cores, 2.4GHz
- **RAM**: 32GB
- **Storage**: 500GB SSD
- **Network**: 1Gbps

### 권장 사양
- **OS**: Ubuntu 22.04 LTS
- **CPU**: 16 cores, 3.0GHz
- **RAM**: 64GB
- **Storage**: 2TB NVMe SSD
- **Network**: 10Gbps

## 🐳 Docker 컨테이너 배포

### 1. Docker Compose 설정

```yaml
# docker-compose.yml
version: '3.8'

services:
  nucleus-server:
    image: nvcr.io/nvidia/omniverse/nucleus-server:latest
    container_name: nucleus-server
    ports:
      - "8080:8080"
      - "8443:8443"
      - "8081:8081"
    environment:
      - NUCLEUS_SERVER_PORT=8080
      - NUCLEUS_SSL_PORT=8443
      - NUCLEUS_WS_PORT=8081
      - POSTGRES_HOST=nucleus-db
      - POSTGRES_DB=nucleus
      - POSTGRES_USER=nucleus_user
      - POSTGRES_PASSWORD=secure_password
    volumes:
      - nucleus-data:/opt/nucleus/data
      - nucleus-config:/opt/nucleus/config
      - ./certs:/opt/nucleus/certs
    depends_on:
      - nucleus-db
    networks:
      - nucleus-network

  nucleus-db:
    image: postgres:14
    container_name: nucleus-db
    environment:
      - POSTGRES_DB=nucleus
      - POSTGRES_USER=nucleus_user
      - POSTGRES_PASSWORD=secure_password
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - nucleus-network

  nginx-proxy:
    image: nginx:alpine
    container_name: nucleus-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs
    depends_on:
      - nucleus-server
    networks:
      - nucleus-network

volumes:
  nucleus-data:
  nucleus-config:
  postgres-data:

networks:
  nucleus-network:
    driver: bridge
```

### 2. 환경 설정 파일

```bash
# .env
NUCLEUS_VERSION=latest
POSTGRES_PASSWORD=your_secure_password_here
SSL_CERT_PATH=./certs/server.crt
SSL_KEY_PATH=./certs/server.key
NUCLEUS_ADMIN_USER=admin
NUCLEUS_ADMIN_PASSWORD=admin_password
```

### 3. SSL 인증서 생성

```bash
#!/bin/bash
# generate-certs.sh

mkdir -p certs

# 자체 서명 인증서 생성 (개발용)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/server.key \
  -out certs/server.crt \
  -subj "/C=KR/ST=Seoul/L=Seoul/O=Company/CN=nucleus.local"

# 인증서 권한 설정
chmod 600 certs/server.key
chmod 644 certs/server.crt

echo "SSL certificates generated successfully!"
```

### 4. Nginx 프록시 설정

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream nucleus-backend {
        server nucleus-server:8080;
    }

    upstream nucleus-websocket {
        server nucleus-server:8081;
    }

    server {
        listen 80;
        server_name nucleus.local;
        return 301 https://$server_name$request_uri;
    }

    server {
        listen 443 ssl http2;
        server_name nucleus.local;

        ssl_certificate /etc/nginx/certs/server.crt;
        ssl_certificate_key /etc/nginx/certs/server.key;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers HIGH:!aNULL:!MD5;

        # HTTP API 프록시
        location /api/ {
            proxy_pass http://nucleus-backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # WebSocket 프록시
        location /ws/ {
            proxy_pass http://nucleus-websocket;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }

        # 정적 파일 서빙
        location / {
            proxy_pass http://nucleus-backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
```

## 🚀 배포 스크립트

### 1. 자동 배포 스크립트

```bash
#!/bin/bash
# deploy-nucleus.sh

set -e

echo "🚀 Starting Nucleus deployment..."

# 환경 변수 확인
if [ ! -f .env ]; then
    echo "❌ .env file not found!"
    exit 1
fi

# Docker 설치 확인
if ! command -v docker &> /dev/null; then
    echo "❌ Docker is not installed!"
    exit 1
fi

if ! command -v docker-compose &> /dev/null; then
    echo "❌ Docker Compose is not installed!"
    exit 1
fi

# SSL 인증서 생성
echo "🔐 Generating SSL certificates..."
./generate-certs.sh

# 데이터베이스 초기화 스크립트
echo "🗄️ Preparing database initialization..."
mkdir -p init-scripts
cat > init-scripts/01-init.sql << EOF
-- Nucleus 데이터베이스 초기화
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 사용자 테이블
CREATE TABLE IF NOT EXISTS users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(255) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(50) DEFAULT 'user',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 프로젝트 테이블
CREATE TABLE IF NOT EXISTS projects (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    owner_id UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 권한 테이블
CREATE TABLE IF NOT EXISTS permissions (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    user_id UUID REFERENCES users(id),
    project_id UUID REFERENCES projects(id),
    permission_type VARCHAR(50) NOT NULL,
    granted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 기본 관리자 사용자 생성
INSERT INTO users (username, email, password_hash, role) 
VALUES ('admin', 'admin@company.com', 'hashed_password', 'admin')
ON CONFLICT (username) DO NOTHING;
EOF

# 컨테이너 실행
echo "🐳 Starting Docker containers..."
docker-compose up -d

# 헬스 체크
echo "🔍 Waiting for services to be ready..."
sleep 30

# Nucleus 서버 상태 확인
if curl -f -s http://localhost:8080/health > /dev/null; then
    echo "✅ Nucleus server is running!"
else
    echo "❌ Nucleus server failed to start!"
    docker-compose logs nucleus-server
    exit 1
fi

# 데이터베이스 연결 확인
if docker-compose exec -T nucleus-db pg_isready -U nucleus_user -d nucleus > /dev/null; then
    echo "✅ Database is ready!"
else
    echo "❌ Database connection failed!"
    docker-compose logs nucleus-db
    exit 1
fi

echo "🎉 Nucleus deployment completed successfully!"
echo "🌐 Access URL: https://nucleus.local"
echo "👤 Admin credentials: admin / admin_password"
```

### 2. 헬스 체크 스크립트

```bash
#!/bin/bash
# health-check.sh

# 서비스 상태 확인
check_service() {
    local service_name=$1
    local url=$2
    
    echo -n "Checking $service_name... "
    if curl -f -s "$url" > /dev/null; then
        echo "✅ OK"
        return 0
    else
        echo "❌ FAILED"
        return 1
    fi
}

echo "🔍 Nucleus Health Check"
echo "======================="

check_service "HTTP API" "http://localhost:8080/health"
check_service "HTTPS API" "https://localhost:8443/health"
check_service "WebSocket" "http://localhost:8081"

# 데이터베이스 연결 확인
echo -n "Checking Database... "
if docker-compose exec -T nucleus-db pg_isready -U nucleus_user -d nucleus > /dev/null 2>&1; then
    echo "✅ OK"
else
    echo "❌ FAILED"
fi

# 디스크 사용량 확인
echo -n "Checking Disk Space... "
DISK_USAGE=$(df -h . | awk 'NR==2 {print $5}' | sed 's/%//')
if [ "$DISK_USAGE" -lt 80 ]; then
    echo "✅ OK ($DISK_USAGE%)"
else
    echo "⚠️  WARNING ($DISK_USAGE%)"
fi

# 메모리 사용량 확인
echo -n "Checking Memory Usage... "
MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100.0}')
if [ "$MEMORY_USAGE" -lt 80 ]; then
    echo "✅ OK ($MEMORY_USAGE%)"
else
    echo "⚠️  WARNING ($MEMORY_USAGE%)"
fi

echo "======================="
echo "Health check completed!"
```

## ☁️ 클라우드 배포 (AWS)

### 1. Terraform 구성

```hcl
# main.tf
provider "aws" {
  region = var.aws_region
}

# VPC 설정
resource "aws_vpc" "nucleus_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "nucleus-vpc"
  }
}

# 서브넷 설정
resource "aws_subnet" "nucleus_subnet" {
  vpc_id                  = aws_vpc.nucleus_vpc.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true

  tags = {
    Name = "nucleus-subnet"
  }
}

# 보안 그룹
resource "aws_security_group" "nucleus_sg" {
  name        = "nucleus-security-group"
  description = "Security group for Nucleus server"
  vpc_id      = aws_vpc.nucleus_vpc.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.admin_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 인스턴스
resource "aws_instance" "nucleus_server" {
  ami           = var.ami_id
  instance_type = var.instance_type
  key_name      = var.key_name
  
  subnet_id                   = aws_subnet.nucleus_subnet.id
  vpc_security_group_ids      = [aws_security_group.nucleus_sg.id]
  associate_public_ip_address = true

  root_block_device {
    volume_type = "gp3"
    volume_size = 100
    encrypted   = true
  }

  user_data = file("${path.module}/user_data.sh")

  tags = {
    Name = "nucleus-server"
  }
}

# RDS 데이터베이스
resource "aws_db_instance" "nucleus_db" {
  identifier             = "nucleus-database"
  engine                 = "postgres"
  engine_version         = "14.9"
  instance_class         = "db.t3.micro"
  allocated_storage      = 20
  storage_encrypted      = true
  
  db_name  = "nucleus"
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.nucleus_sg.id]
  db_subnet_group_name   = aws_db_subnet_group.nucleus_db_subnet_group.name
  
  skip_final_snapshot = true
  
  tags = {
    Name = "nucleus-database"
  }
}
```

### 2. 사용자 데이터 스크립트

```bash
#!/bin/bash
# user_data.sh

yum update -y
yum install -y docker

# Docker 서비스 시작
systemctl start docker
systemctl enable docker

# Docker Compose 설치
curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Nucleus 설정 다운로드 및 실행
cd /opt
git clone https://github.com/your-company/nucleus-config.git
cd nucleus-config

# 환경 변수 설정
export DB_HOST=${db_endpoint}
export DB_PASSWORD=${db_password}

# Nucleus 시작
docker-compose up -d

# 로그 설정
mkdir -p /var/log/nucleus
docker-compose logs -f > /var/log/nucleus/nucleus.log 2>&1 &
```

## 🔗 관련 문서

- [[03-Configuration/SSL-Configuration|SSL 설정 가이드]]
- [[05-Troubleshooting/Installation-Issues|설치 문제 해결]]
- [[06-Scripts-Tools/Deployment-Scripts|배포 자동화 스크립트]]