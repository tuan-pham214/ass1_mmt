# HƯỚNG DẪN CHI TIẾT CODE - HỆ THỐNG P2P FILE SHARING

## MỤC LỤC
1. [Tổng quan kiến trúc](#tổng-quan-kiến-trúc)
2. [File cấu hình (config.py)](#file-cấu-hình-configpy)
3. [Các mô hình dữ liệu (models.py)](#các-mô-hình-dữ-liệu-modelspy)
4. [Phần Server](#phần-server)
5. [Phần Client](#phần-client)
6. [Phần Download](#phần-download)

---

## TỔNG QUAN KIẾN TRÚC

Hệ thống P2P File Sharing gồm 3 thành phần chính:

```
┌─────────────┐         ┌─────────────┐
│   Client A  │         │   Client B  │
│  (Peer 1)   │         │  (Peer 2)   │
└──────┬──────┘         └──────┬──────┘
       │                       │
       │   Đăng ký/Tìm file   │
       │                       │
       └───────┐       ┌───────┘
               ▼       ▼
         ┌─────────────────┐
         │  Central Server │
         │   (Registry)    │
         └─────────────────┘

Client A ←────────────────→ Client B
       Truyền file trực tiếp P2P
```

---

## FILE CẤU HÌNH (config.py)

### Dòng 4-6: Cấu hình Server
```python
DEFAULT_SERVER_HOST = "127.0.0.1"  # Địa chỉ IP server (localhost)
DEFAULT_SERVER_PORT = 9000         # Cổng server lắng nghe
DEFAULT_CLIENT_PORT_START = 9100   # Cổng bắt đầu cho client
```
**Giải thích:**
- Server chạy trên localhost (127.0.0.1) cổng 9000
- Client sẽ dùng cổng từ 9100 trở đi

### Dòng 7-9: Cấu hình kích thước chunk
```python
CHUNK_SIZE = 1024 * 1024           # 1 MB - kích thước mỗi chunk
MAX_CHUNK_SIZE = 4 * 1024 * 1024   # 4 MB - chunk tối đa
MIN_CHUNK_SIZE = 256 * 1024        # 256 KB - chunk tối thiểu
```
**Giải thích:**
- File được chia thành các chunk (khối) để truyền
- Mặc định mỗi chunk 1MB, có thể từ 256KB đến 4MB

### Dòng 10-15: Cấu hình timeout và giới hạn
```python
TIMEOUT_CONNECT = 10.0              # Timeout kết nối: 10 giây
TIMEOUT_READ = 30.0                 # Timeout đọc dữ liệu: 30 giây
TIMEOUT_HEARTBEAT = 5.0             # Timeout heartbeat: 5 giây
MAX_CONCURRENT_UPLOADS = 10         # Tối đa 10 upload đồng thời
MAX_CONCURRENT_DOWNLOADS = 5        # Tối đa 5 download đồng thời
HEARTBEAT_INTERVAL = 30.0           # Gửi heartbeat mỗi 30 giây
```
**Giải thích:**
- Heartbeat: Tín hiệu "còn sống" client gửi cho server định kỳ
- Giới hạn số lượng upload/download đồng thời để tránh quá tải

### Dòng 16-20: Đường dẫn file
```python
REGISTRY_PERSIST_FILE = "registry.json"       # File lưu thông tin registry
CLIENT_CATALOG_FILE = "client_catalog.json"   # File catalog của client
SHARED_DIR_DEFAULT = Path.home() / "p2p_shared"    # Thư mục chia sẻ
DOWNLOAD_DIR_DEFAULT = Path.home() / "p2p_downloads" # Thư mục tải về
BUFFER_SIZE = 64 * 1024                       # Buffer 64KB
```
**Giải thích:**
- `Path.home()`: Thư mục home của user (/home/username)
- File được lưu vào thư mục p2p_shared và tải về p2p_downloads

---

## CÁC MÔ HÌNH DỮ LIỆU (models.py)

### 1. ClientInfo (Dòng 6-18)
```python
@dataclass
class ClientInfo:
    hostname: str      # Tên máy client
    ip: str           # Địa chỉ IP
    port: int         # Cổng lắng nghe
    last_seen: float  # Thời điểm gặp lần cuối (timestamp)
```
**Giải thích:**
- `@dataclass`: Decorator tự động tạo `__init__`, `__repr__`, etc.
- `last_seen`: Dùng để kiểm tra client còn online không
- `to_dict()`: Chuyển object thành dictionary để lưu JSON
- `from_dict()`: Tạo object từ dictionary

**Ví dụ:**
```python
client = ClientInfo(
    hostname="peer1",
    ip="192.168.1.100",
    port=9100,
    last_seen=1699000000.0
)
```

### 2. FileInfo (Dòng 20-32)
```python
@dataclass
class FileInfo:
    fname: str              # Tên file công khai (trên mạng P2P)
    lname: str              # Tên file local (trên máy)
    size: Optional[int]     # Kích thước file (bytes)
    sha256: Optional[str]   # Mã hash SHA256 để kiểm tra toàn vẹn
```
**Giải thích:**
- `fname` (file name): Tên khi chia sẻ, vd: "video.mp4"
- `lname` (local name): Tên thật trên máy, vd: "my_video.mp4"
- `sha256`: Mã hash dùng để kiểm tra file có bị hỏng không
- `Optional[int]`: Có thể có giá trị int hoặc None

### 3. PeerInfo (Dòng 34-52)
```python
@dataclass
class PeerInfo:
    hostname: str              # Tên peer
    ip: str                   # IP của peer
    port: int                 # Cổng của peer
    rtt: Optional[float]      # Round Trip Time (độ trễ ping)
    bandwidth: Optional[float] # Băng thông
```
**Giải thích:**
- `rtt` (Round Trip Time): Thời gian ping, thấp = nhanh hơn
- Hàm `score()` tính điểm peer (RTT thấp + bandwidth cao = điểm cao)

**Công thức điểm:**
```python
def score(self) -> float:
    rtt_score = 1.0 / (self.rtt + 0.001)  # RTT càng nhỏ, điểm càng cao
    bw_score = self.bandwidth if self.bandwidth else 1.0
    return rtt_score * bw_score
```

### 4. Message và Response (Dòng 54-89)
```python
@dataclass
class Message:
    op: str                    # Loại thao tác: GET, PUBLISH, QUERY...
    data: Dict[str, Any]       # Dữ liệu kèm theo

@dataclass
class Response:
    ok: bool                   # Thành công hay thất bại
    data: Optional[Dict]       # Dữ liệu trả về (nếu thành công)
    error: Optional[str]       # Thông báo lỗi (nếu thất bại)
```
**Giải thích:**
- Dùng để giao tiếp giữa client-server và peer-peer
- `to_json()`: Chuyển thành chuỗi JSON để gửi qua mạng
- `from_json()`: Parse chuỗi JSON nhận được

---

## PHẦN SERVER

### File: server_main.py

#### Lớp P2PServer (Dòng 9-34)

**Dòng 10-15: Khởi tạo server**
```python
def __init__(self, host: str, port: int, persist_path: Optional[Path] = None):
    self.host = host                              # Địa chỉ IP server
    self.port = port                              # Cổng server
    self.registry = Registry(persist_path)        # Tạo registry quản lý client
    self.protocol = ServerProtocol(self.registry) # Tạo protocol xử lý request
    self.server: Optional[asyncio.Server] = None  # Server chưa khởi động
```
**Giải thích:**
- `Registry`: Quản lý danh sách client và file
- `ServerProtocol`: Xử lý các request từ client
- `Optional[asyncio.Server]`: Server có thể None hoặc là Server object

**Dòng 17-28: Khởi động server**
```python
async def start(self):
    # Tạo server lắng nghe kết nối
    self.server = await asyncio.start_server(
        self.protocol.handle_client,  # Hàm xử lý mỗi client kết nối
        self.host,                    # IP lắng nghe
        self.port                     # Cổng lắng nghe
    )

    # Lấy địa chỉ thực tế server đang chạy
    addr = self.server.sockets[0].getsockname() if self.server.sockets else (self.host, self.port)
    print(f"Server started on {addr[0]}:{addr[1]}")

    # Chạy server mãi mãi (cho đến khi dừng)
    async with self.server:
        await self.server.serve_forever()
```
**Giải thích:**
- `asyncio.start_server()`: Tạo TCP server bất đồng bộ
- `handle_client`: Callback được gọi khi có client kết nối
- `serve_forever()`: Chạy vô hạn, chờ kết nối

**Dòng 30-33: Dừng server**
```python
async def stop(self):
    if self.server:
        self.server.close()              # Đóng server, không nhận kết nối mới
        await self.server.wait_closed()  # Đợi đóng hoàn toàn
```

#### Lớp AdminShell (Dòng 35-94)

**Mục đích:** Shell admin để quản lý server từ dòng lệnh

**Dòng 43-78: Vòng lặp xử lý lệnh**
```python
async def _command_loop(self):
    print("Admin shell ready. Commands: discover <hostname>, ping <hostname>, quit")

    async def process_commands():
        # Đọc lệnh từ stdin (không blocking)
        try:
            line = await asyncio.get_event_loop().run_in_executor(None, sys.stdin.readline)
        except EOFError:
            return None
```
**Giải thích:**
- `run_in_executor()`: Chạy stdin.readline() trong thread pool
- Vì readline() blocking, phải chạy trong executor để không block event loop

**Dòng 56-72: Parse và thực thi lệnh**
```python
        line = line.strip()  # Xóa khoảng trắng đầu/cuối
        if not line:
            return await process_commands()  # Dòng trống, đọc tiếp

        if line == 'quit':
            self.running = False
            return None

        # Tách lệnh và tham số
        parts = line.split(None, 1)  # split(None, 1) tách theo whitespace, tối đa 2 phần
        cmd = parts[0] if parts else ''
        arg = parts[1] if len(parts) > 1 else ''

        # Xử lý từng lệnh
        if cmd == 'discover' and arg:
            await self._cmd_discover(arg)  # Xem file của client
        elif cmd == 'ping' and arg:
            await self._cmd_ping(arg)      # Kiểm tra client còn sống không
        else:
            print(f"Unknown command: {line}")
```

**Dòng 80-94: Các lệnh admin**
```python
async def _cmd_discover(self, hostname: str):
    # Lấy danh sách file của client
    files = self.registry.get_client_files(hostname)
    if files:
        print(f"Files on {hostname}:")
        list(map(lambda f: print(f"  - {f}"), files))  # In từng file
    else:
        print(f"No files found on {hostname}")

async def _cmd_ping(self, hostname: str):
    # Kiểm tra client còn online không
    alive = self.registry.is_client_alive(hostname)
    if alive:
        client = self.registry.clients.get(hostname)
        print(f"{hostname} is alive at {client.ip}:{client.port}")
    else:
        print(f"{hostname} is not responding or not registered")
```

#### Hàm main() (Dòng 96-121)

**Dòng 97-107: Parse tham số dòng lệnh**
```python
import argparse

parser = argparse.ArgumentParser(description='P2P File Sharing Server')
parser.add_argument('--host', default=DEFAULT_SERVER_HOST, help='Server host')
parser.add_argument('--port', type=int, default=DEFAULT_SERVER_PORT, help='Server port')
parser.add_argument('--persist', default='registry.json', help='Registry persist file')

args = parser.parse_args()

persist_path = Path(args.persist)
server = P2PServer(args.host, args.port, persist_path)
```
**Giải thích:**
- `argparse`: Module parse tham số command line
- `--host`, `--port`: Tham số tùy chọn, có giá trị mặc định
- Ví dụ chạy: `python server_main.py --host 0.0.0.0 --port 8000`

**Dòng 109-118: Chạy server và admin shell song song**
```python
shell = AdminShell(server.registry)

# Tạo 2 task chạy song song
server_task = asyncio.create_task(server.start())  # Task 1: Server
shell_task = asyncio.create_task(shell.run())      # Task 2: Admin shell

try:
    # Chạy cả 2 task, đợi đến khi một trong hai kết thúc
    await asyncio.gather(server_task, shell_task, return_exceptions=True)
except KeyboardInterrupt:
    print("\nShutting down...")
    await server.stop()
```
**Giải thích:**
- `create_task()`: Tạo task chạy đồng thời
- `gather()`: Chạy nhiều coroutine song song
- `return_exceptions=True`: Không dừng nếu một task lỗi

### File: registry.py

#### Khởi tạo Registry (Dòng 8-13)
```python
def __init__(self, persist_path: Optional[Path] = None):
    self.clients: Dict[str, ClientInfo] = {}    # {hostname: ClientInfo}
    self.file_index: Dict[str, Set[str]] = {}   # {fname: {hostname1, hostname2, ...}}
    self.persist_path = persist_path or Path(REGISTRY_PERSIST_FILE)
    self._load()  # Load dữ liệu từ file (nếu có)
```
**Giải thích:**
- `clients`: Dictionary lưu thông tin tất cả client
- `file_index`: Map từ tên file → set các hostname có file đó
- Ví dụ: `{"video.mp4": {"peer1", "peer2", "peer3"}}`

#### Load dữ liệu (Dòng 15-31)
```python
def _load(self):
    if not self.persist_path.exists():  # File chưa tồn tại
        return

    try:
        with open(self.persist_path, 'r') as f:
            data = json.load(f)  # Parse JSON

            # Chuyển dict thành ClientInfo objects
            self.clients = dict(map(
                lambda item: (item[0], ClientInfo.from_dict(item[1])),
                data.get('clients', {}).items()
            ))

            # Chuyển list thành set cho file_index
            self.file_index = dict(map(
                lambda item: (item[0], set(item[1])),
                data.get('file_index', {}).items()
            ))
    except (json.JSONDecodeError, OSError):
        pass  # Lỗi thì bỏ qua, dùng dữ liệu trống
```
**Giải thích:**
- `map()`: Áp dụng function cho từng item
- `lambda item: (item[0], ClientInfo.from_dict(item[1]))`:
  - `item[0]`: hostname (key)
  - `item[1]`: dict data (value) → convert thành ClientInfo

#### Lưu dữ liệu (Dòng 33-49)
```python
def _save(self):
    # Chuyển ClientInfo objects thành dict
    data = {
        'clients': dict(map(
            lambda item: (item[0], item[1].to_dict()),
            self.clients.items()
        )),
        # Chuyển set thành list (JSON không hỗ trợ set)
        'file_index': dict(map(
            lambda item: (item[0], list(item[1])),
            self.file_index.items()
        ))
    }

    try:
        with open(self.persist_path, 'w') as f:
            json.dump(data, f, indent=2)  # Lưu JSON với indent đẹp
    except OSError:
        pass
```

#### Đăng ký client (Dòng 51-61)
```python
def register_client(self, hostname: str, ip: str, port: int, files: List[str]) -> bool:
    # Lưu thông tin client
    self.clients[hostname] = ClientInfo(
        hostname=hostname,
        ip=ip,
        port=port,
        last_seen=time.time()  # Timestamp hiện tại
    )

    # Thêm tất cả file vào index
    def add_file(fname):
        if fname not in self.file_index:
            self.file_index[fname] = set()  # Tạo set mới nếu chưa có
        self.file_index[fname].add(hostname)  # Thêm hostname vào set

    list(map(add_file, files))  # Áp dụng add_file cho từng file
    self._save()  # Lưu vào disk
    return True
```
**Giải thích:**
- Client gửi REGISTER với danh sách file → server lưu vào registry
- `time.time()`: Timestamp Unix (số giây từ 1/1/1970)

#### Publish file mới (Dòng 63-73)
```python
def publish_file(self, hostname: str, fname: str) -> bool:
    if hostname not in self.clients:
        return False  # Client chưa đăng ký

    # Thêm file vào index
    if fname not in self.file_index:
        self.file_index[fname] = set()

    self.file_index[fname].add(hostname)
    self.clients[hostname].last_seen = time.time()  # Cập nhật last_seen
    self._save()
    return True
```

#### Tìm peer có file (Dòng 75-92)
```python
def query_file(self, fname: str) -> List[PeerInfo]:
    if fname not in self.file_index:
        return []  # Không có peer nào có file

    hostnames = self.file_index[fname]  # Set các hostname

    # Chuyển hostname thành PeerInfo
    def to_peer_info(hostname: str) -> Optional[PeerInfo]:
        client = self.clients.get(hostname)
        if client:
            return PeerInfo(
                hostname=client.hostname,
                ip=client.ip,
                port=client.port
            )
        return None

    # Map và filter None
    peers = list(filter(None, map(to_peer_info, hostnames)))
    return peers
```
**Giải thích:**
- Client hỏi "ai có file video.mp4?"
- Server trả về list PeerInfo của tất cả peer có file đó

#### Heartbeat (Dòng 94-100)
```python
def heartbeat(self, hostname: str) -> bool:
    if hostname not in self.clients:
        return False

    # Cập nhật thời gian gặp lần cuối
    self.clients[hostname].last_seen = time.time()
    self._save()
    return True
```
**Giải thích:**
- Client gửi HEARTBEAT định kỳ để báo "tôi còn sống"
- Server cập nhật `last_seen`

#### Kiểm tra client còn sống (Dòng 111-115)
```python
def is_client_alive(self, hostname: str, timeout: float = 60.0) -> bool:
    client = self.clients.get(hostname)
    if not client:
        return False
    # So sánh thời gian hiện tại với last_seen
    return (time.time() - client.last_seen) < timeout
```
**Giải thích:**
- Nếu last_seen cách đây > 60s → client đã offline

---

## PHẦN CLIENT

### File: client_main.py

#### Lớp P2PClient - Khởi tạo (Dòng 18-47)

```python
def __init__(
    self,
    hostname: str,        # Tên client này
    server_host: str,     # IP của server
    server_port: int,     # Port của server
    client_port: int,     # Port client lắng nghe (để peer kết nối)
    shared_dir: Path,     # Thư mục chứa file chia sẻ
    download_dir: Path    # Thư mục lưu file tải về
):
    self.hostname = hostname
    self.server_host = server_host
    self.server_port = server_port
    self.client_port = client_port
    self.shared_dir = shared_dir
    self.download_dir = download_dir

    ensure_dir(shared_dir)      # Tạo thư mục nếu chưa có
    ensure_dir(download_dir)

    # Tạo uploader để cho peer tải file từ mình
    self.uploader = P2PUploader(shared_dir, client_port)

    # Tạo downloader để tải file từ peer
    self.downloader = P2PDownloader(download_dir)

    # Catalog: danh sách file đang chia sẻ
    self.catalog: Dict[str, FileInfo] = {}
    self.catalog_path = shared_dir / CLIENT_CATALOG_FILE

    self.heartbeat_task: Optional[asyncio.Task] = None
    self.running = True

    self._load_catalog()  # Load catalog từ file
```

#### Load/Save Catalog (Dòng 49-72)

**Load catalog:**
```python
def _load_catalog(self):
    if not self.catalog_path.exists():
        return

    try:
        with open(self.catalog_path, 'r') as f:
            data = json.load(f)
            # Chuyển dict → FileInfo objects
            self.catalog = dict(map(
                lambda item: (item[0], FileInfo.from_dict(item[1])),
                data.items()
            ))
    except (json.JSONDecodeError, OSError):
        pass
```

**Save catalog:**
```python
def _save_catalog(self):
    try:
        with open(self.catalog_path, 'w') as f:
            # Chuyển FileInfo objects → dict
            data = dict(map(
                lambda item: (item[0], item[1].to_dict()),
                self.catalog.items()
            ))
            json.dump(data, f, indent=2)
    except OSError:
        pass
```

#### Khởi động client (Dòng 74-84)

```python
async def start(self):
    # Khởi động uploader (server P2P cho peer kết nối)
    await self.uploader.start()

    # Lấy danh sách file trong catalog
    files = list(self.catalog.keys())

    # Đăng ký với server
    await self._register_with_server(files)

    # Bắt đầu heartbeat loop
    self.heartbeat_task = asyncio.create_task(self._heartbeat_loop())

    print(f"Client '{self.hostname}' started on port {self.client_port}")
    print(f"Shared directory: {self.shared_dir}")
    print(f"Download directory: {self.download_dir}")
```

#### Đăng ký với server (Dòng 98-125)

```python
async def _register_with_server(self, files: list):
    try:
        # Kết nối đến server
        reader, writer = await asyncio.wait_for(
            asyncio.open_connection(self.server_host, self.server_port),
            timeout=TIMEOUT_CONNECT  # 10 giây
        )

        # Tạo request REGISTER
        request = {
            'op': 'REGISTER',
            'hostname': self.hostname,
            'port': self.client_port,  # Port để peer kết nối
            'files': files             # Danh sách file đang có
        }

        # Gửi request (JSON + newline)
        await write_json_line(writer, request)

        # Nhận response từ server
        response = await read_json_line(reader)

        if response and response.get('ok'):
            print(f"Registered with server")
        else:
            error = response.get('error', 'Unknown error') if response else 'No response'
            print(f"Registration failed: {error}")

        # Đóng kết nối
        writer.close()
        await writer.wait_closed()

    except (asyncio.TimeoutError, OSError) as e:
        print(f"Server connection failed: {e}")
```
**Giải thích:**
- `asyncio.open_connection()`: Mở kết nối TCP async
- `wait_for(..., timeout=10)`: Timeout nếu quá 10s
- `write_json_line()`: Ghi JSON + `\n` vào stream
- `read_json_line()`: Đọc một dòng JSON từ stream

#### Heartbeat loop (Dòng 127-156)

```python
async def _heartbeat_loop(self):
    # Dùng đệ quy thay vì while True
    async def heartbeat_recursive():
        if not self.running:
            return  # Dừng nếu client tắt

        # Đợi 30 giây
        await asyncio.sleep(HEARTBEAT_INTERVAL)

        try:
            # Kết nối server
            reader, writer = await asyncio.wait_for(
                asyncio.open_connection(self.server_host, self.server_port),
                timeout=TIMEOUT_CONNECT
            )

            # Gửi HEARTBEAT
            request = {'op': 'HEARTBEAT', 'hostname': self.hostname}
            await write_json_line(writer, request)

            # Nhận response (không xử lý gì)
            await read_json_line(reader)

            writer.close()
            await writer.wait_closed()

        except (asyncio.TimeoutError, OSError):
            pass  # Bỏ qua lỗi, thử lại sau 30s

        # Gọi đệ quy
        await heartbeat_recursive()

    try:
        await heartbeat_recursive()
    except asyncio.CancelledError:
        pass  # Task bị cancel khi client stop
```
**Giải thích:**
- Dùng đệ quy thay vì while loop (functional programming style)
- Mỗi 30s gửi 1 heartbeat cho server
- Nếu lỗi thì bỏ qua, thử lại lần sau

#### Publish file (Dòng 158-197)

```python
async def publish(self, lname: str, fname: str):
    # lname: tên file local (vd: my_video.mp4)
    # fname: tên file public (vd: video.mp4)

    local_path = self.shared_dir / lname

    # Kiểm tra file có tồn tại không
    if not local_path.exists():
        print(f"Error: Local file '{lname}' not found in {self.shared_dir}")
        return

    # Kiểm tra file có trong thư mục shared không (bảo mật)
    if not is_safe_path(self.shared_dir, local_path):
        print(f"Error: Path '{lname}' is outside shared directory")
        return

    try:
        # Kết nối server
        reader, writer = await asyncio.wait_for(
            asyncio.open_connection(self.server_host, self.server_port),
            timeout=TIMEOUT_CONNECT
        )

        # Gửi PUBLISH request
        request = {
            'op': 'PUBLISH',
            'hostname': self.hostname,
            'fname': fname
        }
        await write_json_line(writer, request)

        # Nhận response
        response = await read_json_line(reader)

        if response and response.get('ok'):
            # Thêm vào catalog
            self.catalog[fname] = FileInfo(fname=fname, lname=lname)

            # Thêm vào uploader (để peer có thể tải)
            self.uploader.add_file(fname, lname)

            # Lưu catalog
            self._save_catalog()
            print(f"Published '{fname}' -> '{lname}'")
        else:
            error = response.get('error', 'Unknown error') if response else 'No response'
            print(f"Publish failed: {error}")

        writer.close()
        await writer.wait_closed()

    except (asyncio.TimeoutError, OSError) as e:
        print(f"Server connection failed: {e}")
```
**Giải thích:**
- Client muốn chia sẻ file → gọi publish
- Gửi thông tin file cho server
- Server lưu vào registry
- Uploader cập nhật để sẵn sàng cho peer tải

#### Fetch file (Dòng 199-245)

```python
async def fetch(self, fname: str):
    try:
        # 1. Hỏi server: ai có file này?
        reader, writer = await asyncio.wait_for(
            asyncio.open_connection(self.server_host, self.server_port),
            timeout=TIMEOUT_CONNECT
        )

        request = {'op': 'QUERY', 'fname': fname}
        await write_json_line(writer, request)

        response = await read_json_line(reader)

        writer.close()
        await writer.wait_closed()

        if not response or not response.get('ok'):
            error = response.get('error', 'Unknown error') if response else 'No response'
            print(f"Query failed: {error}")
            return

        # 2. Nhận danh sách peer có file
        peers_data = response.get('data', {}).get('peers', [])

        if not peers_data:
            print(f"No peers available file: {fname}")
            return

        # Chuyển dict → PeerInfo objects
        peers = list(map(PeerInfo.from_dict, peers_data))

        print(f"Found {len(peers)} peer(s) with '{fname}'")

        # 3. Chọn peer tốt nhất (ping, bandwidth)
        selected_peer = await select_peer_interactive(peers)

        if not selected_peer:
            print("No suitable peer found")
            return

        print(f"Selected peer: {selected_peer.hostname} (RTT: {selected_peer.rtt:.3f}s)")

        # 4. Tải file từ peer đã chọn
        result = await self.downloader.download_file(selected_peer, fname)

        if result:
            print(f"File saved to: {result}")
        else:
            print("Download failed")

    except (asyncio.TimeoutError, OSError) as e:
        print(f"Server connection failed: {e}")
```
**Giải thích:**
QUY TRÌNH TẢI FILE:
1. Client hỏi server: "Ai có file video.mp4?"
2. Server trả về danh sách peer: [peer1, peer2, peer3]
3. Client ping tất cả peer, chọn peer nhanh nhất
4. Client kết nối trực tiếp peer đó, tải file P2P

#### Client Shell (Dòng 247-286)

```python
class ClientShell:
    def __init__(self, client: P2PClient):
        self.client = client

    async def run(self):
        print("\nClient shell ready. Commands: publish <lname> <fname>, fetch <fname>, quit")
        await self._command_loop()

    async def _command_loop(self):
        try:
            # Đọc lệnh từ stdin
            line = await asyncio.get_event_loop().run_in_executor(None, sys.stdin.readline)
        except EOFError:
            return

        if not line:
            return

        line = line.strip()

        if not line:
            return await self._command_loop()  # Dòng trống, đọc tiếp

        if line == 'quit':
            return  # Thoát shell

        # Parse lệnh
        parts = line.split(None, 2)
        cmd = parts[0] if parts else ''

        # Xử lý lệnh
        if cmd == 'publish' and len(parts) == 3:
            lname = parts[1]  # Tên file local
            fname = parts[2]  # Tên file public
            await self.client.publish(lname, fname)
        elif cmd == 'fetch' and len(parts) == 2:
            fname = parts[1]
            await self.client.fetch(fname)
        else:
            print(f"Unknown command: {line}")

        return await self._command_loop()  # Đọc lệnh tiếp theo
```
**Giải thích:**
- Shell tương tác cho user
- Lệnh `publish my_video.mp4 video.mp4`: Chia sẻ file local
- Lệnh `fetch video.mp4`: Tải file từ peer

---

## PHẦN DOWNLOAD

### File: p2p_downloader.py

#### Khởi tạo Downloader (Dòng 12-15)
```python
class P2PDownloader:
    def __init__(self, download_dir: Path):
        self.download_dir = download_dir
        ensure_dir(download_dir)  # Tạo thư mục nếu chưa có
```

#### Download file chính (Dòng 17-88)

**Dòng 17-28: Khởi tạo download**
```python
async def download_file(
    self,
    peer: PeerInfo,      # Peer sẽ tải file từ
    fname: str,          # Tên file
    chunk_size: int = CHUNK_SIZE,  # Kích thước chunk (1MB)
    resume: bool = True  # Có tiếp tục download nếu bị gián đoạn không
) -> Optional[Path]:
    dest_path = self.download_dir / fname
    offset = 0

    # Nếu file đã tồn tại, tiếp tục từ byte cuối
    if resume and dest_path.exists():
        offset = dest_path.stat().st_size  # Lấy kích thước file hiện tại
```
**Giải thích:**
- Resume: Nếu tải dở, lần sau tiếp tục thay vì tải lại
- `offset`: Vị trí byte bắt đầu tải

**Dòng 30-37: Kết nối đến peer**
```python
    try:
        # Kết nối TCP đến peer
        reader, writer = await asyncio.wait_for(
            asyncio.open_connection(peer.ip, peer.port),
            timeout=TIMEOUT_CONNECT
        )
    except (asyncio.TimeoutError, OSError) as e:
        print(f"Connection failed: {e}")
        return None
```

**Dòng 39-57: Gửi request GET**
```python
    try:
        # Tạo request GET file
        request = {
            'op': 'GET',
            'fname': fname,
            'offset': offset,      # Bắt đầu từ byte nào
            'chunk_size': chunk_size
        }
        await write_json_line(writer, request)

        # Nhận response
        response = await read_json_line(reader, timeout=TIMEOUT_READ)

        if not response or not response.get('ok'):
            error = response.get('error', 'Unknown error') if response else 'No response'
            print(f"Download failed: {error}")
            return None

        # Lấy thông tin file
        file_size = response.get('size', 0)        # Kích thước file
        file_sha256 = response.get('file_sha256', '')  # Hash để kiểm tra

        print(f"Downloading {fname} ({file_size} bytes) from {peer.hostname}")
```

**Dòng 60-66: Nhận data**
```python
        start_time = time.perf_counter()  # Bắt đầu đo thời gian

        # Nhận các chunk từ peer
        success = await self._receive_file_chunks(reader, dest_path, offset)

        elapsed = time.perf_counter() - start_time  # Thời gian tải

        if not success:
            print("Download failed during transfer")
            return None
```

**Dòng 68-76: Kiểm tra integrity**
```python
        # Tính hash của file vừa tải
        actual_hash = await compute_file_sha256(dest_path)

        # So sánh với hash từ peer
        if actual_hash != file_sha256:
            print(f"Integrity check failed: expected {file_sha256}, got {actual_hash}")
            dest_path.unlink(missing_ok=True)  # Xóa file lỗi
            return None

        # Tính throughput (tốc độ tải)
        throughput = file_size / (elapsed * 1024 * 1024) if elapsed > 0 else 0
        print(f"Download complete: {fname} ({throughput:.2f} MB/s)")

        return dest_path
```
**Giải thích:**
- SHA256: Mã hash 256-bit, dùng để kiểm tra file có bị hỏng không
- Nếu hash khác → file bị lỗi → xóa đi
- Throughput: MB/s (megabytes per second)

#### Nhận file chunks (Dòng 90-105)

```python
async def _receive_file_chunks(self, reader: asyncio.StreamReader, dest_path: Path, offset: int) -> bool:
    # Chọn mode: append (nối tiếp) hoặc write (ghi mới)
    mode = 'ab' if offset > 0 else 'wb'

    try:
        with open(dest_path, mode) as f:
            # Dùng đệ quy để nhận chunk
            async def receive_loop():
                # Đọc một chunk (với hash để verify)
                chunk = await read_chunk_with_hash(reader)

                if chunk is None:
                    return True  # Hết file

                f.write(chunk)  # Ghi chunk vào file

                return await receive_loop()  # Đọc chunk tiếp theo

            return await receive_loop()

    except Exception:
        return False
```
**Giải thích:**
- `'ab'`: Append binary (nối tiếp vào file)
- `'wb'`: Write binary (ghi mới)
- Mỗi chunk có hash riêng để verify ngay

---

## TÓM TẮT FLOW HOẠT ĐỘNG

### 1. Khởi động hệ thống

```
1. Chạy Server:
   python -m server.server_main --port 9000
   → Server lắng nghe trên port 9000

2. Chạy Client A:
   python -m client.client_main --hostname peerA --port 9100
   → Client đăng ký với server
   → Uploader lắng nghe trên port 9100

3. Chạy Client B:
   python -m client.client_main --hostname peerB --port 9101
   → Client đăng ký với server
   → Uploader lắng nghe trên port 9101
```

### 2. Chia sẻ file (Client A)

```
Client A> publish my_video.mp4 video.mp4

Flow:
1. Client A kiểm tra file "my_video.mp4" tồn tại trong thư mục shared
2. Gửi PUBLISH request đến Server
3. Server lưu vào registry: file_index["video.mp4"] = {"peerA"}
4. Client A thêm vào catalog và uploader
```

### 3. Tải file (Client B)

```
Client B> fetch video.mp4

Flow:
1. Client B gửi QUERY request đến Server: "Ai có video.mp4?"
2. Server trả về: [{"hostname": "peerA", "ip": "127.0.0.1", "port": 9100}]
3. Client B ping peerA để đo RTT
4. Client B kết nối trực tiếp peerA (P2P):
   - Gửi GET request với offset=0
   - PeerA gửi file qua các chunk 1MB
   - Client B nhận chunk, verify hash từng chunk
5. Sau khi nhận hết, Client B verify SHA256 toàn bộ file
6. File được lưu vào thư mục downloads
```

### 4. Resume download

```
Nếu download bị gián đoạn:
- File đã tải 50MB / 100MB
- Lần tải lại, Client B gửi GET với offset=50MB
- PeerA chỉ gửi 50MB còn lại
- Client B nối tiếp vào file cũ (mode 'ab')
```

---

## CÁC KHÁI NIỆM QUAN TRỌNG

### 1. Asyncio
```python
async def my_function():
    await asyncio.sleep(1)  # Không block, CPU làm việc khác

# Khác với:
def blocking_function():
    time.sleep(1)  # Block, CPU chờ
```

### 2. TCP Stream
```python
reader, writer = await asyncio.open_connection(host, port)

# Gửi data
writer.write(b"Hello")
await writer.drain()  # Đợi gửi xong

# Nhận data
data = await reader.read(1024)  # Đọc tối đa 1024 bytes
```

### 3. JSON Protocol
```
Message format:
{"op": "REGISTER", "hostname": "peerA", "port": 9100}\n
{"op": "QUERY", "fname": "video.mp4"}\n

Response format:
{"ok": true, "data": {...}}\n
{"ok": false, "error": "File not found"}\n
```

### 4. Hash SHA256
```python
import hashlib

# Tính hash file
sha256 = hashlib.sha256()
with open('file.mp4', 'rb') as f:
    while True:
        chunk = f.read(65536)
        if not chunk:
            break
        sha256.update(chunk)

file_hash = sha256.hexdigest()  # "a1b2c3d4..."
```

---

## CÂU HỎI THƯỜNG GẶP KHI TRÌNH BÀY

### Q1: Tại sao dùng asyncio thay vì threads?
**A:**
- Asyncio nhẹ hơn, 1 process có thể chạy hàng nghìn coroutine
- Thread nặng, tối đa vài trăm threads
- Asyncio tránh race condition (không cần lock)

### Q2: Tại sao cần heartbeat?
**A:**
- Để server biết client còn online không
- Nếu client offline, không trả kết quả nó cho user khác
- Timeout 60s: không nhận heartbeat > 60s → coi như offline

### Q3: Tại sao chia file thành chunk?
**A:**
- File lớn (vài GB) không thể load hết vào RAM
- Chia chunk → tải từng phần, verify từng phần
- Tiết kiệm băng thông nếu lỗi (chỉ tải lại chunk lỗi)

### Q4: Resume download hoạt động thế nào?
**A:**
- Lưu offset = số byte đã tải
- Lần sau gửi GET với offset
- Peer gửi từ vị trí offset
- Client nối tiếp vào file (append mode)

### Q5: Peer selection hoạt động thế nào?
**A:**
- Ping tất cả peer, đo RTT
- Tính score = 1/(RTT + 0.001) * bandwidth
- Chọn peer có score cao nhất (RTT thấp, bandwidth cao)

### Q6: Làm sao đảm bảo file không bị hỏng?
**A:**
- 2 lớp verify:
  1. Hash từng chunk (ngay khi nhận)
  2. Hash toàn bộ file (sau khi tải xong)
- Nếu hash sai → xóa file, báo lỗi

---

## DEMO THỰC TẾ

### Bước 1: Chạy Server
```bash
cd /home/user/BLT1---Mang-may-tinh-
python -m server.server_main --port 9000
```

### Bước 2: Chạy Client 1
```bash
# Terminal mới
python -m client.client_main --hostname peer1 --port 9100 --shared-dir ./sample_files/peer1 --download-dir ./downloads/peer1
```

### Bước 3: Chạy Client 2
```bash
# Terminal mới
python -m client.client_main --hostname peer2 --port 9101 --shared-dir ./sample_files/peer2 --download-dir ./downloads/peer2
```

### Bước 4: Publish file (Client 1)
```
publish file1.txt file1.txt
```

### Bước 5: Fetch file (Client 2)
```
fetch file1.txt
```

### Bước 6: Kiểm tra server
```
# Tại terminal server
discover peer1
ping peer2
```

---

## KẾT LUẬN

Hệ thống P2P File Sharing này bao gồm:

1. **Central Server (Registry):**
   - Quản lý danh sách client
   - Index file (ai có file gì)
   - Heartbeat tracking

2. **Client:**
   - Đăng ký với server
   - Publish file để chia sẻ
   - Query và tải file từ peer
   - Uploader: Cho peer khác tải file
   - Downloader: Tải file từ peer

3. **P2P Transfer:**
   - Kết nối trực tiếp giữa peer
   - Truyền file qua chunk
   - Verify integrity bằng SHA256
   - Hỗ trợ resume download

4. **Tính năng nâng cao:**
   - Peer selection (chọn peer tốt nhất)
   - Concurrent transfer
   - Error handling và retry
   - Persistence (lưu registry, catalog)

Hy vọng tài liệu này giúp bạn hiểu rõ và tự tin trình bày với thầy!
