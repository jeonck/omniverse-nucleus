# 문제 해결 및 트러블슈팅 가이드

## 🔍 일반적인 문제와 해결방법

### 1. 연결 문제

#### 문제: Nucleus 서버에 연결할 수 없음
```bash
# 증상
Connection refused: omniverse://nucleus.company.com
HTTP 500 Internal Server Error
```

**해결방법:**
```bash
# 1. 서버 상태 확인
curl -I https://nucleus.company.com/health

# 2. 네트워크 연결 확인
ping nucleus.company.com
telnet nucleus.company.com 8080

# 3. 방화벽 설정 확인
sudo ufw status
sudo iptables -L

# 4. 서비스 재시작
sudo systemctl restart nucleus
sudo systemctl status nucleus
```

#### 문제: SSL 인증서 오류
```bash
# 증상
SSL certificate verify failed
certificate has expired
```

**해결방법:**
```bash
# 1. 인증서 갱신
sudo certbot renew --nginx

# 2. 인증서 상태 확인
openssl x509 -in /etc/nginx/ssl/nucleus.crt -text -noout

# 3. 자체 서명 인증서 재생성
./generate-certs.sh

# 4. Nginx 설정 테스트
sudo nginx -t
sudo systemctl reload nginx
```

### 2. 성능 문제

#### 문제: 파일 업로드가 느림
```python
# 진단 스크립트
import time
import requests
from pathlib import Path

def diagnose_upload_performance(file_path, nucleus_url, auth_token):
    """업로드 성능 진단"""
    file_size = Path(file_path).stat().st_size
    
    print(f"파일 크기: {file_size / (1024*1024):.2f} MB")
    
    headers = {'Authorization': f'Bearer {auth_token}'}
    
    # 업로드 시간 측정
    start_time = time.time()
    
    with open(file_path, 'rb') as f:
        response = requests.post(
            f'{nucleus_url}/api/v1/files/upload',
            headers=headers,
            files={'file': f}
        )
    
    end_time = time.time()
    upload_time = end_time - start_time
    
    if response.status_code == 200:
        speed_mbps = (file_size / (1024*1024)) / upload_time
        print(f"업로드 속도: {speed_mbps:.2f} MB/s")
        
        if speed_mbps < 1.0:
            print("⚠️ 업로드 속도가 느립니다. 네트워크 또는 서버 성능을 확인하세요.")
    else:
        print(f"❌ 업로드 실패: {response.status_code}")
```

**최적화 방법:**
```bash
# 1. Nginx 업로드 설정 최적화
# /etc/nginx/nginx.conf
client_max_body_size 10G;
client_body_buffer_size 1M;
client_body_timeout 300s;
client_header_timeout 300s;

# 2. 커널 네트워크 파라미터 튜닝
echo 'net.core.rmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 87380 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 134217728' >> /etc/sysctl.conf
sysctl -p

# 3. 디스크 I/O 최적화
echo 'deadline' > /sys/block/sda/queue/scheduler
echo '2048' > /sys/block/sda/queue/read_ahead_kb
```

#### 문제: 데이터베이스 성능 저하
```sql
-- 성능 진단 쿼리
-- 느린 쿼리 확인
SELECT 
    query,
    mean_time,
    calls,
    total_time,
    mean_time * calls as total_time_calc
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;

-- 인덱스 사용률 확인
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes 
ORDER BY idx_scan DESC;

-- 테이블 통계 확인
SELECT 
    schemaname,
    tablename,
    n_tup_ins,
    n_tup_upd,
    n_tup_del,
    n_live_tup,
    n_dead_tup
FROM pg_stat_user_tables 
ORDER BY n_live_tup DESC;
```

**최적화 방법:**
```sql
-- 1. 인덱스 최적화
CREATE INDEX CONCURRENTLY idx_files_project_created 
ON files(project_id, created_at);

CREATE INDEX CONCURRENTLY idx_files_path_gin 
ON files USING gin(to_tsvector('english', path));

-- 2. 통계 정보 업데이트
ANALYZE;

-- 3. 불필요한 데이터 정리
DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '90 days';
VACUUM ANALYZE audit_logs;

-- 4. 파티셔닝 설정 (대용량 테이블용)
CREATE TABLE files_2024 PARTITION OF files
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

### 3. 동기화 문제

#### 문제: 실시간 동기화가 작동하지 않음
```python
# 동기화 상태 진단 스크립트
import websocket
import json
import threading
import time

class SyncDiagnostics:
    def __init__(self, nucleus_url, auth_token):
        self.nucleus_url = nucleus_url
        self.auth_token = auth_token
        self.connected = False
        self.messages_received = 0
        self.last_heartbeat = None
    
    def test_websocket_connection(self):
        """WebSocket 연결 테스트"""
        try:
            ws_url = f"{self.nucleus_url.replace('https://', 'wss://')}/ws/sync"
            headers = [f"Authorization: Bearer {self.auth_token}"]
            
            def on_message(ws, message):
                self.messages_received += 1
                data = json.loads(message)
                if data.get('type') == 'heartbeat':
                    self.last_heartbeat = time.time()
                print(f"📨 메시지 수신: {data.get('type', 'unknown')}")
            
            def on_error(ws, error):
                print(f"❌ WebSocket 오류: {error}")
            
            def on_close(ws, close_status_code, close_msg):
                print(f"🔌 연결 종료: {close_status_code}")
                self.connected = False
            
            def on_open(ws):
                print("✅ WebSocket 연결 성공")
                self.connected = True
                # 테스트 메시지 송신
                ws.send(json.dumps({'type': 'ping'}))
            
            ws = websocket.WebSocketApp(
                ws_url,
                header=headers,
                on_open=on_open,
                on_message=on_message,
                on_error=on_error,
                on_close=on_close
            )
            
            # 10초간 연결 테스트
            ws.run_forever()
            
        except Exception as e:
            print(f"❌ WebSocket 테스트 실패: {e}")
    
    def diagnose_sync_issues(self):
        """동기화 문제 진단"""
        print("🔍 동기화 진단 시작...")
        
        # 1. WebSocket 연결 테스트
        print("1. WebSocket 연결 테스트")
        threading.Thread(target=self.test_websocket_connection).start()
        
        time.sleep(10)
        
        # 2. 결과 분석
        if not self.connected:
            print("❌ WebSocket 연결 실패 - 서버 또는 네트워크 문제")
        elif self.messages_received == 0:
            print("⚠️ 메시지 수신 없음 - 서버 설정 확인 필요")
        elif self.last_heartbeat is None:
            print("⚠️ 하트비트 없음 - 연결 유지 문제")
        else:
            print("✅ 동기화 연결 정상")

# 사용 예제
diagnostics = SyncDiagnostics(
    nucleus_url='https://nucleus.company.com',
    auth_token='your_token_here'
)
diagnostics.diagnose_sync_issues()
```

**해결방법:**
```bash
# 1. WebSocket 프록시 설정 확인
# /etc/nginx/sites-available/nucleus
location /ws/ {
    proxy_pass http://nucleus_backend;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_read_timeout 86400s;  # 24시간
    proxy_send_timeout 86400s;
}

# 2. 방화벽에서 WebSocket 포트 열기
sudo ufw allow 8081/tcp

# 3. Nucleus 설정에서 WebSocket 활성화
# nucleus-config.yaml
websocket:
  enabled: true
  port: 8081
  heartbeat_interval: 30
  max_connections: 1000
```

### 4. 권한 문제

#### 문제: 파일 접근 권한 거부
```python
# 권한 진단 스크립트
def diagnose_permissions(nucleus_client, user_id, file_id):
    """사용자 권한 진단"""
    try:
        # 1. 사용자 정보 확인
        user_info = nucleus_client._request('GET', f'/users/{user_id}')
        print(f"사용자 역할: {user_info.json().get('role')}")
        
        # 2. 파일 정보 및 권한 확인
        file_info = nucleus_client._request('GET', f'/files/{file_id}')
        if file_info.status_code == 200:
            file_data = file_info.json()
            print(f"파일 소유자: {file_data.get('owner')}")
            print(f"프로젝트 ID: {file_data.get('project_id')}")
        elif file_info.status_code == 403:
            print("❌ 파일 읽기 권한 없음")
        elif file_info.status_code == 404:
            print("❌ 파일이 존재하지 않음")
        
        # 3. 프로젝트 권한 확인
        project_id = file_data.get('project_id')
        if project_id:
            permissions = nucleus_client._request(
                'GET', f'/projects/{project_id}/permissions/{user_id}'
            )
            if permissions.status_code == 200:
                perms = permissions.json()
                print(f"프로젝트 권한: {perms.get('permissions', [])}")
            else:
                print("❌ 프로젝트 권한 없음")
        
    except Exception as e:
        print(f"❌ 권한 진단 실패: {e}")

# 권한 수정 스크립트
def fix_permissions(nucleus_client, user_id, project_id, permissions):
    """권한 수정"""
    permission_data = {
        'user_id': user_id,
        'project_id': project_id,
        'permissions': permissions
    }
    
    response = nucleus_client._request(
        'POST', '/permissions',
        json=permission_data
    )
    
    if response.status_code in [201, 200]:
        print("✅ 권한 수정 완료")
    else:
        print(f"❌ 권한 수정 실패: {response.text}")
```

### 5. 로그 분석 및 디버깅

#### 시스템 로그 분석
```bash
#!/bin/bash
# log_analyzer.sh

echo "🔍 Nucleus 로그 분석 시작..."

# 1. 최근 에러 로그 확인
echo "=== 최근 에러 로그 ==="
tail -100 /var/log/nucleus/nucleus.log | grep -i error

# 2. 연결 실패 로그
echo "=== 연결 실패 로그 ==="
grep "connection refused\|timeout\|failed to connect" /var/log/nucleus/nucleus.log | tail -20

# 3. 권한 관련 로그
echo "=== 권한 관련 로그 ==="
grep -i "permission denied\|unauthorized\|forbidden" /var/log/nucleus/nucleus.log | tail -20

# 4. 성능 관련 로그
echo "=== 느린 쿼리 로그 ==="
grep "slow query\|execution time" /var/log/nucleus/nucleus.log | tail -10

# 5. 시스템 리소스 확인
echo "=== 시스템 리소스 ==="
echo "CPU 사용률:"
top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1

echo "메모리 사용률:"
free -m | awk 'NR==2{printf "%.1f%%\n", $3*100/$2}'

echo "디스크 사용률:"
df -h / | awk 'NR==2 {print $5}'

# 6. 네트워크 연결 상태
echo "=== 네트워크 연결 ==="
netstat -tuln | grep -E ":8080|:8443|:8081"

# 7. 데이터베이스 연결 확인
echo "=== 데이터베이스 연결 ==="
sudo -u postgres psql -d nucleus -c "SELECT count(*) as active_connections FROM pg_stat_activity WHERE state = 'active';"
```

#### 상세 디버깅 설정
```yaml
# nucleus-config.yaml - 디버깅 모드
logging:
  level: "DEBUG"
  file: "/var/log/nucleus/nucleus-debug.log"
  max_size: "500MB"
  max_files: 5
  format: "json"
  
  # 상세 로깅 모듈
  modules:
    api: "DEBUG"
    database: "INFO"
    websocket: "DEBUG"
    file_operations: "DEBUG"
    authentication: "DEBUG"

performance:
  # 성능 모니터링 활성화
  metrics_enabled: true
  slow_query_threshold: 1000  # ms
  request_timeout: 30000  # ms
  
  # 프로파일링
  profiling:
    enabled: true
    sample_rate: 0.1  # 10% 샘플링
    output_dir: "/var/log/nucleus/profiles"
```

## 🚨 응급 복구 절차

### 1. 서비스 전체 다운 시
```bash
#!/bin/bash
# emergency_recovery.sh

echo "🚨 응급 복구 시작..."

# 1. 서비스 상태 확인
systemctl is-active nucleus
systemctl is-active postgresql
systemctl is-active nginx
systemctl is-active redis

# 2. 기본 서비스 재시작
echo "⚙️ 기본 서비스 재시작..."
systemctl restart postgresql
sleep 5
systemctl restart redis
sleep 5
systemctl restart nucleus
sleep 10
systemctl restart nginx

# 3. 서비스 상태 재확인
if systemctl is-active --quiet nucleus; then
    echo "✅ Nucleus 서비스 복구 완료"
else
    echo "❌ Nucleus 서비스 복구 실패"
    systemctl status nucleus
fi

# 4. 헬스 체크
curl -f http://localhost:8080/health
if [ $? -eq 0 ]; then
    echo "✅ 헬스 체크 통과"
else
    echo "❌ 헬스 체크 실패"
fi

# 5. 기본 연결 테스트
python3 -c "
import requests
try:
    response = requests.get('http://localhost:8080/api/v1/status', timeout=5)
    print(f'API 응답: {response.status_code}')
except Exception as e:
    print(f'API 연결 실패: {e}')
"
```

### 2. 데이터베이스 복구
```sql
-- database_recovery.sql

-- 1. 데이터베이스 연결 확인
SELECT version();

-- 2. 테이블 무결성 확인
SELECT 
    schemaname,
    tablename,
    has_table_privilege(current_user, schemaname||'.'||tablename, 'SELECT') as can_read
FROM pg_tables 
WHERE schemaname = 'public';

-- 3. 인덱스 재구성
REINDEX DATABASE nucleus;

-- 4. 통계 정보 업데이트
ANALYZE;

-- 5. 연결 풀 초기화
SELECT pg_terminate_backend(pid) 
FROM pg_stat_activity 
WHERE datname = 'nucleus' 
  AND pid <> pg_backend_pid()
  AND state = 'idle';

-- 6. 백업에서 복구 (필요시)
-- pg_restore -d nucleus /backup/nucleus_backup.dump
```

### 3. 파일 시스템 복구
```bash
#!/bin/bash
# filesystem_recovery.sh

NUCLEUS_DATA="/opt/nucleus/data"
BACKUP_DIR="/backup/nucleus"

echo "🗂️ 파일 시스템 복구 시작..."

# 1. 파일 시스템 검사
echo "1. 파일 시스템 무결성 검사..."
fsck -n /dev/sda1  # 읽기 전용 검사

# 2. 권한 복구
echo "2. 파일 권한 복구..."
chown -R nucleus:nucleus "$NUCLEUS_DATA"
chmod -R 755 "$NUCLEUS_DATA"
chmod -R 644 "$NUCLEUS_DATA"/*.usd

# 3. 손상된 파일 검사
echo "3. 손상된 파일 검사..."
find "$NUCLEUS_DATA" -name "*.usd" -exec file {} \; | grep -v "ASCII text"

# 4. 백업에서 복구 (필요시)
if [ -d "$BACKUP_DIR" ] && [ "$(ls -A $BACKUP_DIR)" ]; then
    echo "4. 백업에서 파일 복구..."
    read -p "백업에서 복구하시겠습니까? (y/N): " confirm
    if [[ $confirm == [yY] ]]; then
        rsync -av --progress "$BACKUP_DIR/" "$NUCLEUS_DATA/"
        echo "✅ 백업 복구 완료"
    fi
fi

# 5. 디스크 공간 확인
echo "5. 디스크 공간 확인..."
df -h "$NUCLEUS_DATA"
```

## 📊 성능 모니터링

### 1. 실시간 모니터링 스크립트
```python
#!/usr/bin/env python3
# performance_monitor.py

import psutil
import requests
import time
import json
from datetime import datetime

class NucleusMonitor:
    def __init__(self, nucleus_url, check_interval=60):
        self.nucleus_url = nucleus_url
        self.check_interval = check_interval
        self.metrics_history = []
    
    def collect_metrics(self):
        """시스템 메트릭 수집"""
        metrics = {
            'timestamp': datetime.now().isoformat(),
            'cpu_percent': psutil.cpu_percent(interval=1),
            'memory_percent': psutil.virtual_memory().percent,
            'disk_usage': psutil.disk_usage('/').percent,
            'network_io': psutil.net_io_counters()._asdict(),
            'nucleus_status': self.check_nucleus_health()
        }
        
        return metrics
    
    def check_nucleus_health(self):
        """Nucleus 서비스 상태 확인"""
        try:
            response = requests.get(
                f'{self.nucleus_url}/health',
                timeout=5
            )
            return {
                'status': 'healthy' if response.status_code == 200 else 'unhealthy',
                'response_time': response.elapsed.total_seconds(),
                'status_code': response.status_code
            }
        except Exception as e:
            return {
                'status': 'down',
                'error': str(e)
            }
    
    def analyze_performance(self, metrics):
        """성능 분석 및 알림"""
        alerts = []
        
        if metrics['cpu_percent'] > 80:
            alerts.append({
                'type': 'HIGH_CPU',
                'value': metrics['cpu_percent'],
                'threshold': 80,
                'severity': 'WARNING'
            })
        
        if metrics['memory_percent'] > 85:
            alerts.append({
                'type': 'HIGH_MEMORY',
                'value': metrics['memory_percent'],
                'threshold': 85,
                'severity': 'CRITICAL'
            })
        
        if metrics['disk_usage'] > 90:
            alerts.append({
                'type': 'HIGH_DISK',
                'value': metrics['disk_usage'],
                'threshold': 90,
                'severity': 'CRITICAL'
            })
        
        nucleus_status = metrics['nucleus_status']['status']
        if nucleus_status != 'healthy':
            alerts.append({
                'type': 'SERVICE_DOWN',
                'value': nucleus_status,
                'severity': 'CRITICAL'
            })
        
        return alerts
    
    def send_alerts(self, alerts):
        """알림 전송"""
        for alert in alerts:
            print(f"🚨 {alert['severity']}: {alert['type']} - {alert.get('value')}")
            
            # 이메일, Slack 등으로 알림 전송 로직 추가
            # self.send_email_alert(alert)
            # self.send_slack_alert(alert)
    
    def run_monitoring(self):
        """모니터링 실행"""
        print("📊 Nucleus 성능 모니터링 시작...")
        
        while True:
            try:
                metrics = self.collect_metrics()
                self.metrics_history.append(metrics)
                
                # 최근 24시간 데이터만 유지
                if len(self.metrics_history) > 1440:  # 24시간 * 60분
                    self.metrics_history.pop(0)
                
                alerts = self.analyze_performance(metrics)
                if alerts:
                    self.send_alerts(alerts)
                
                # 상태 출력
                print(f"⏰ {metrics['timestamp']}")
                print(f"   CPU: {metrics['cpu_percent']:.1f}%")
                print(f"   Memory: {metrics['memory_percent']:.1f}%")
                print(f"   Disk: {metrics['disk_usage']:.1f}%")
                print(f"   Nucleus: {metrics['nucleus_status']['status']}")
                
                time.sleep(self.check_interval)
                
            except KeyboardInterrupt:
                print("\n모니터링 종료")
                break
            except Exception as e:
                print(f"❌ 모니터링 오류: {e}")
                time.sleep(30)

if __name__ == '__main__':
    monitor = NucleusMonitor('http://localhost:8080')
    monitor.run_monitoring()
```

## 🔗 관련 문서

- [[02-Installation/Installation-Guide|설치 가이드]]
- [[03-Configuration/Performance-Tuning|성능 최적화]]
- [[06-Scripts-Tools/Monitoring-Scripts|모니터링 스크립트]]