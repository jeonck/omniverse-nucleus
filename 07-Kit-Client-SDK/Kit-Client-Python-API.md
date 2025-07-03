# Kit Client Library Python SDK

## 🐍 Omniverse Kit Client Library Python API

Omniverse Kit Client Library는 Python 애플리케이션에서 Nucleus 서버와 직접 상호작용할 수 있는 강력한 도구입니다. 이 라이브러리를 통해 파일 시스템 작업, 실시간 협업, 인증 등을 프로그래밍 방식으로 처리할 수 있습니다.

## 📚 주요 기능

### 1. 기본 설정 및 초기화

```python
import omni.client
import asyncio
from typing import Optional, List, Tuple

class OmniverseClient:
    def __init__(self, server_url: str = None):
        self.server_url = server_url
        self.initialized = False
    
    def initialize(self) -> bool:
        """Client Library 초기화"""
        try:
            # 클라이언트 라이브러리 초기화
            result = omni.client.initialize()
            if result:
                self.initialized = True
                print("✅ Omniverse Client Library 초기화 완료")
                
                # 제품 정보 설정
                omni.client.set_product_info(
                    name="Custom Omniverse App",
                    version="1.0.0",
                    extra="Python SDK Integration"
                )
                
                # 로그 레벨 설정
                omni.client.set_log_level(omni.client.LogLevel.INFO)
                
                return True
            else:
                print("❌ Omniverse Client Library 초기화 실패")
                return False
        except Exception as e:
            print(f"❌ 초기화 중 오류: {e}")
            return False
    
    def shutdown(self):
        """Client Library 종료"""
        if self.initialized:
            omni.client.shutdown()
            self.initialized = False
            print("🔒 Omniverse Client Library 종료")

# 사용 예제
client = OmniverseClient("omniverse://nucleus.company.com")
if client.initialize():
    # 작업 수행
    pass
client.shutdown()
```

### 2. 인증 및 연결 관리

```python
class AuthenticationManager:
    def __init__(self):
        self.auth_callbacks = []
        self.connection_status = {}
    
    def setup_authentication(self):
        """인증 콜백 설정"""
        
        def auth_callback(server_url: str):
            """인증 콜백 함수"""
            print(f"🔐 인증 요청: {server_url}")
            
            # API 토큰 사용
            api_token = os.getenv('OMNIVERSE_API_TOKEN')
            if api_token:
                return ("$omni-api-token", api_token)
            
            # 사용자명/비밀번호 사용
            username = os.getenv('OMNIVERSE_USERNAME')
            password = os.getenv('OMNIVERSE_PASSWORD')
            if username and password:
                return (username, password)
            
            # 인증 실패
            return None
        
        def connection_status_callback(url: str, status: omni.client.ConnectionStatus):
            """연결 상태 콜백"""
            self.connection_status[url] = status
            status_name = status.name
            
            if status == omni.client.ConnectionStatus.CONNECTED:
                print(f"✅ 연결 성공: {url}")
            elif status == omni.client.ConnectionStatus.CONNECT_ERROR:
                print(f"❌ 연결 오류: {url}")
            elif status == omni.client.ConnectionStatus.AUTH_FAILED:
                print(f"🔑 인증 실패: {url}")
            elif status == omni.client.ConnectionStatus.DISCONNECTED:
                print(f"🔌 연결 해제: {url}")
            else:
                print(f"📡 연결 상태 변경: {url} - {status_name}")
        
        # 콜백 등록
        auth_reg = omni.client.register_authentication_callback(auth_callback)
        status_reg = omni.client.register_connection_status_callback(connection_status_callback)
        
        self.auth_callbacks.append(auth_reg)
        self.auth_callbacks.append(status_reg)
        
        return auth_reg, status_reg
    
    def cleanup(self):
        """인증 콜백 정리"""
        for callback in self.auth_callbacks:
            del callback
        self.auth_callbacks.clear()

# 사용 예제
auth_manager = AuthenticationManager()
auth_reg, status_reg = auth_manager.setup_authentication()

# 작업 완료 후 정리
auth_manager.cleanup()
```

### 3. 파일 시스템 작업

```python
class FileSystemManager:
    def __init__(self, base_url: str):
        self.base_url = base_url
        omni.client.push_base_url(base_url)
    
    async def create_folder(self, folder_path: str) -> bool:
        """폴더 생성"""
        try:
            result = await omni.client.create_folder_async(folder_path)
            if result == omni.client.Result.OK:
                print(f"📁 폴더 생성 완료: {folder_path}")
                return True
            else:
                print(f"❌ 폴더 생성 실패: {folder_path} - {result.name}")
                return False
        except Exception as e:
            print(f"❌ 폴더 생성 오류: {e}")
            return False
    
    async def upload_file(self, local_path: str, remote_path: str, message: str = "") -> bool:
        """파일 업로드"""
        try:
            # 로컬 파일 읽기
            with open(local_path, 'rb') as f:
                content = f.read()
            
            # 원격 파일 쓰기
            result = await omni.client.write_file_async(remote_path, content, message)
            
            if result == omni.client.Result.OK:
                print(f"📤 파일 업로드 완료: {local_path} -> {remote_path}")
                return True
            else:
                print(f"❌ 파일 업로드 실패: {result.name}")
                return False
        except Exception as e:
            print(f"❌ 파일 업로드 오류: {e}")
            return False
    
    async def download_file(self, remote_path: str, local_path: str) -> bool:
        """파일 다운로드"""
        try:
            result, version, content = await omni.client.read_file_async(remote_path)
            
            if result == omni.client.Result.OK:
                # 파일 저장
                with open(local_path, 'wb') as f:
                    f.write(memoryview(content).tobytes())
                
                print(f"📥 파일 다운로드 완료: {remote_path} -> {local_path}")
                return True
            else:
                print(f"❌ 파일 다운로드 실패: {result.name}")
                return False
        except Exception as e:
            print(f"❌ 파일 다운로드 오류: {e}")
            return False
    
    async def list_directory(self, directory_path: str) -> Optional[List[dict]]:
        """디렉토리 목록 조회"""
        try:
            result, entries = await omni.client.list_async(directory_path)
            
            if result == omni.client.Result.OK:
                file_list = []
                for entry in entries:
                    file_info = {
                        'name': entry.relative_path,
                        'size': entry.size,
                        'modified_time': entry.modified_time,
                        'is_folder': bool(entry.flags & omni.client.ItemFlags.CAN_HAVE_CHILDREN),
                        'is_file': bool(entry.flags & omni.client.ItemFlags.READABLE_FILE),
                        'version': entry.version,
                        'hash': entry.hash if hasattr(entry, 'hash') else None
                    }
                    file_list.append(file_info)
                
                print(f"📋 디렉토리 목록: {len(file_list)}개 항목")
                return file_list
            else:
                print(f"❌ 디렉토리 목록 조회 실패: {result.name}")
                return None
        except Exception as e:
            print(f"❌ 디렉토리 목록 조회 오류: {e}")
            return None
    
    async def copy_file(self, src_path: str, dst_path: str, overwrite: bool = False) -> bool:
        """파일 복사"""
        try:
            behavior = omni.client.CopyBehavior.OVERWRITE if overwrite else omni.client.CopyBehavior.ERROR_IF_EXISTS
            result = await omni.client.copy_file_async(src_path, dst_path, behavior)
            
            if result == omni.client.Result.OK:
                print(f"📋 파일 복사 완료: {src_path} -> {dst_path}")
                return True
            else:
                print(f"❌ 파일 복사 실패: {result.name}")
                return False
        except Exception as e:
            print(f"❌ 파일 복사 오류: {e}")
            return False
    
    async def move_file(self, src_path: str, dst_path: str, overwrite: bool = False) -> bool:
        """파일 이동"""
        try:
            behavior = omni.client.CopyBehavior.OVERWRITE if overwrite else omni.client.CopyBehavior.ERROR_IF_EXISTS
            result, copied = await omni.client.move_file_async(src_path, dst_path, behavior)
            
            if result == omni.client.Result.OK:
                action = "복사 후 삭제" if copied else "서버 내 이동"
                print(f"📦 파일 이동 완료 ({action}): {src_path} -> {dst_path}")
                return True
            else:
                print(f"❌ 파일 이동 실패: {result.name}")
                return False
        except Exception as e:
            print(f"❌ 파일 이동 오류: {e}")
            return False
    
    async def delete_file(self, file_path: str) -> bool:
        """파일 삭제"""
        try:
            result = await omni.client.delete_async(file_path)
            
            if result == omni.client.Result.OK:
                print(f"🗑️ 파일 삭제 완료: {file_path}")
                return True
            else:
                print(f"❌ 파일 삭제 실패: {result.name}")
                return False
        except Exception as e:
            print(f"❌ 파일 삭제 오류: {e}")
            return False

# 사용 예제
async def main():
    fs_manager = FileSystemManager("omniverse://nucleus.company.com/Projects/MyProject")
    
    # 폴더 생성
    await fs_manager.create_folder("assets/models")
    
    # 파일 업로드
    await fs_manager.upload_file("local_model.usd", "assets/models/character.usd", "Initial upload")
    
    # 디렉토리 목록 조회
    files = await fs_manager.list_directory("assets")
    for file_info in files or []:
        print(f"📄 {file_info['name']} ({file_info['size']} bytes)")
    
    # 파일 다운로드
    await fs_manager.download_file("assets/models/character.usd", "downloaded_character.usd")

if __name__ == "__main__":
    asyncio.run(main())
```

### 4. 실시간 협업 및 Live Updates

```python
class LiveCollaborationManager:
    def __init__(self):
        self.subscriptions = []
        self.live_callbacks = []
    
    def setup_live_updates(self):
        """실시간 업데이트 설정"""
        
        def live_update_callback(
            update_type: omni.client.LiveUpdateType,
            result: omni.client.Result,
            url: str,
            from_client_id: int,
            server_time: int,
            content_size: int
        ):
            """라이브 업데이트 콜백"""
            if result == omni.client.Result.OK:
                if update_type == omni.client.LiveUpdateType.REMOTE:
                    print(f"🔄 원격 업데이트 수신: {url} (클라이언트 {from_client_id})")
                elif update_type == omni.client.LiveUpdateType.LOCAL:
                    print(f"✅ 로컬 업데이트 확인: {url}")
                elif update_type == omni.client.LiveUpdateType.MORE:
                    print(f"⏭️ 추가 업데이트 대기: {url}")
                
                # 라이브 업데이트 처리
                self.process_live_updates()
            else:
                print(f"❌ 라이브 업데이트 오류: {result.name}")
        
        # 라이브 업데이트 콜백 등록
        registration = omni.client.live_register_queued_callback(live_update_callback)
        self.live_callbacks.append(registration)
        
        return registration
    
    def process_live_updates(self):
        """라이브 업데이트 처리"""
        try:
            omni.client.live_process()
        except Exception as e:
            print(f"❌ 라이브 업데이트 처리 오류: {e}")
    
    async def subscribe_to_file_changes(self, file_path: str):
        """파일 변경사항 구독"""
        
        def stat_callback(result: omni.client.Result, entry):
            """초기 파일 정보 콜백"""
            if result == omni.client.Result.OK:
                print(f"📄 파일 정보: {file_path}")
                print(f"   크기: {entry.size} bytes")
                print(f"   수정일: {entry.modified_time}")
                print(f"   버전: {entry.version}")
        
        def subscribe_callback(
            result: omni.client.Result, 
            event: omni.client.ListEvent, 
            entry
        ):
            """파일 변경 콜백"""
            if result == omni.client.Result.OK:
                event_name = event.name
                if event == omni.client.ListEvent.UPDATED:
                    print(f"📝 파일 업데이트: {file_path}")
                elif event == omni.client.ListEvent.CREATED:
                    print(f"📄 파일 생성: {file_path}")
                elif event == omni.client.ListEvent.DELETED:
                    print(f"🗑️ 파일 삭제: {file_path}")
                elif event == omni.client.ListEvent.LOCKED:
                    print(f"🔒 파일 잠금: {file_path}")
                elif event == omni.client.ListEvent.UNLOCKED:
                    print(f"🔓 파일 잠금 해제: {file_path}")
                else:
                    print(f"📡 파일 이벤트: {file_path} - {event_name}")
        
        try:
            subscription = omni.client.stat_subscribe_with_callback(
                file_path,
                stat_callback,
                subscribe_callback
            )
            self.subscriptions.append(subscription)
            print(f"👁️ 파일 변경사항 구독 시작: {file_path}")
            return subscription
        except Exception as e:
            print(f"❌ 파일 구독 오류: {e}")
            return None
    
    async def subscribe_to_folder_changes(self, folder_path: str):
        """폴더 변경사항 구독"""
        
        def list_callback(result: omni.client.Result, entries):
            """초기 폴더 목록 콜백"""
            if result == omni.client.Result.OK:
                print(f"📁 폴더 목록: {folder_path} ({len(entries)}개 항목)")
                for entry in entries:
                    print(f"   📄 {entry.relative_path}")
        
        def subscribe_callback(
            result: omni.client.Result,
            event: omni.client.ListEvent,
            entry
        ):
            """폴더 변경 콜백"""
            if result == omni.client.Result.OK:
                event_name = event.name
                item_name = entry.relative_path
                
                if event == omni.client.ListEvent.CREATED:
                    print(f"➕ 항목 추가: {folder_path}/{item_name}")
                elif event == omni.client.ListEvent.DELETED:
                    print(f"➖ 항목 삭제: {folder_path}/{item_name}")
                elif event == omni.client.ListEvent.UPDATED:
                    print(f"🔄 항목 업데이트: {folder_path}/{item_name}")
                else:
                    print(f"📡 폴더 이벤트: {folder_path} - {event_name}")
        
        try:
            subscription = omni.client.list_subscribe_with_callback(
                folder_path,
                list_callback,
                subscribe_callback
            )
            self.subscriptions.append(subscription)
            print(f"👁️ 폴더 변경사항 구독 시작: {folder_path}")
            return subscription
        except Exception as e:
            print(f"❌ 폴더 구독 오류: {e}")
            return None
    
    def cleanup(self):
        """리소스 정리"""
        for subscription in self.subscriptions:
            del subscription
        self.subscriptions.clear()
        
        for callback in self.live_callbacks:
            del callback
        self.live_callbacks.clear()

# 사용 예제
async def collaboration_example():
    collab_manager = LiveCollaborationManager()
    
    # 라이브 업데이트 설정
    collab_manager.setup_live_updates()
    
    # 파일 변경사항 구독
    await collab_manager.subscribe_to_file_changes("omniverse://nucleus.company.com/Projects/MyProject/scene.usd")
    
    # 폴더 변경사항 구독
    await collab_manager.subscribe_to_folder_changes("omniverse://nucleus.company.com/Projects/MyProject/assets")
    
    # 실시간 업데이트 대기
    try:
        while True:
            await asyncio.sleep(1)
            collab_manager.process_live_updates()
    except KeyboardInterrupt:
        print("🛑 협업 모니터링 중단")
    finally:
        collab_manager.cleanup()

if __name__ == "__main__":
    asyncio.run(collaboration_example())
```

### 5. 체크포인트 및 버전 관리

```python
class VersionManager:
    def __init__(self):
        pass
    
    async def create_checkpoint(self, file_path: str, comment: str, force: bool = False) -> Optional[str]:
        """체크포인트 생성"""
        try:
            result, checkpoint_id = await omni.client.create_checkpoint_async(
                file_path, comment, force
            )
            
            if result == omni.client.Result.OK:
                print(f"📌 체크포인트 생성: {file_path}")
                print(f"   ID: {checkpoint_id}")
                print(f"   코멘트: {comment}")
                return checkpoint_id
            else:
                print(f"❌ 체크포인트 생성 실패: {result.name}")
                return None
        except Exception as e:
            print(f"❌ 체크포인트 생성 오류: {e}")
            return None
    
    async def list_checkpoints(self, file_path: str) -> Optional[List[dict]]:
        """체크포인트 목록 조회"""
        try:
            result, entries = await omni.client.list_checkpoints_async(file_path)
            
            if result == omni.client.Result.OK:
                checkpoints = []
                for entry in entries:
                    checkpoint_info = {
                        'id': entry.relative_path,
                        'comment': entry.comment if hasattr(entry, 'comment') else '',
                        'created_time': entry.created_time if hasattr(entry, 'created_time') else entry.modified_time,
                        'created_by': entry.created_by if hasattr(entry, 'created_by') else '',
                        'size': entry.size
                    }
                    checkpoints.append(checkpoint_info)
                
                print(f"📋 체크포인트 목록: {len(checkpoints)}개")
                return checkpoints
            else:
                print(f"❌ 체크포인트 목록 조회 실패: {result.name}")
                return None
        except Exception as e:
            print(f"❌ 체크포인트 목록 조회 오류: {e}")
            return None
    
    def create_branch_url(self, base_url: str, branch: str, checkpoint: int = None) -> str:
        """브랜치 URL 생성"""
        if checkpoint is not None:
            query = omni.client.make_query_from_branch_and_checkpoint(branch, checkpoint)
        else:
            query = f"branch={branch}"
        
        parsed_url = omni.client.break_url(base_url)
        return omni.client.make_url(
            scheme=parsed_url.scheme,
            user=parsed_url.user,
            host=parsed_url.host,
            port=parsed_url.port,
            path=parsed_url.path,
            query=query,
            fragment=parsed_url.fragment
        )
    
    def parse_branch_and_checkpoint(self, url: str) -> Tuple[Optional[str], Optional[int]]:
        """URL에서 브랜치와 체크포인트 추출"""
        parsed_url = omni.client.break_url(url)
        if parsed_url.query:
            return omni.client.get_branch_and_checkpoint_from_query(parsed_url.query)
        return None, None

# 사용 예제
async def version_management_example():
    version_manager = VersionManager()
    
    file_path = "omniverse://nucleus.company.com/Projects/MyProject/scene.usd"
    
    # 체크포인트 생성
    checkpoint_id = await version_manager.create_checkpoint(
        file_path, 
        "Added new lighting setup",
        force=False
    )
    
    # 체크포인트 목록 조회
    checkpoints = await version_manager.list_checkpoints(file_path)
    if checkpoints:
        for cp in checkpoints:
            print(f"📌 {cp['id']}: {cp['comment']}")
    
    # 브랜치 URL 생성
    branch_url = version_manager.create_branch_url(file_path, "experimental", 123)
    print(f"🌿 브랜치 URL: {branch_url}")
    
    # 브랜치와 체크포인트 파싱
    branch, checkpoint = version_manager.parse_branch_and_checkpoint(branch_url)
    print(f"🔍 파싱 결과: 브랜치={branch}, 체크포인트={checkpoint}")

if __name__ == "__main__":
    asyncio.run(version_management_example())
```

### 6. 캐시 및 로컬 파일 관리

```python
class CacheManager:
    def __init__(self):
        self.cached_files = {}
    
    async def open_cached_file(self, remote_url: str, download: bool = True) -> Optional[str]:
        """캐시된 파일 열기"""
        try:
            result, handle, local_path = await omni.client.open_cached_file_async(
                remote_url, download
            )
            
            if result == omni.client.Result.OK:
                self.cached_files[remote_url] = handle
                print(f"📂 캐시된 파일 열기: {remote_url}")
                print(f"   로컬 경로: {local_path}")
                return local_path
            else:
                print(f"❌ 캐시된 파일 열기 실패: {result.name}")
                return None
        except Exception as e:
            print(f"❌ 캐시된 파일 열기 오류: {e}")
            return None
    
    def close_cached_file(self, remote_url: str):
        """캐시된 파일 닫기"""
        if remote_url in self.cached_files:
            handle = self.cached_files[remote_url]
            omni.client.close_cached_file(handle)
            del self.cached_files[remote_url]
            print(f"🔒 캐시된 파일 닫기: {remote_url}")
    
    def cleanup(self):
        """모든 캐시된 파일 정리"""
        for remote_url in list(self.cached_files.keys()):
            self.close_cached_file(remote_url)

# 사용 예제
async def cache_example():
    cache_manager = CacheManager()
    
    try:
        # 캐시된 파일 열기
        local_path = await cache_manager.open_cached_file(
            "omniverse://nucleus.company.com/Projects/MyProject/scene.usd"
        )
        
        if local_path:
            # 로컬 파일로 작업
            with open(local_path, 'r') as f:
                content = f.read()
                print(f"📄 파일 크기: {len(content)} 문자")
        
    finally:
        cache_manager.cleanup()

if __name__ == "__main__":
    asyncio.run(cache_example())
```

## 🔗 관련 문서

- [[01-Overview/Overview|개요]]
- [[02-Installation/Installation-Guide|설치 가이드]]
- [[03-Configuration/Configuration-Guide|설정 가이드]]
- [[04-API-Integration/API-Guide|API 통합]]
- [[05-Troubleshooting/Common-Issues|문제 해결]]
- [[06-Scripts-Tools/Utility-Scripts|스크립트 도구]]

---

이 Kit Client Library Python SDK 가이드는 Omniverse Nucleus와의 직접적인 상호작용을 위한 모든 필수 기능을 다룹니다. 실시간 협업, 버전 관리, 파일 시스템 작업 등을 프로그래밍 방식으로 구현할 수 있습니다.