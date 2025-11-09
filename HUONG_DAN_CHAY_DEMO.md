# HƯỚNG DẪN CHẠY DEMO HỆ THỐNG P2P FILE SHARING

## Tổng quan
Hệ thống P2P File Sharing cho phép nhiều máy tính chia sẻ và tải file trực tiếp với nhau thông qua một server trung tâm.

## Yêu cầu hệ thống
- Python 3.11 trở lên
- pip package manager

---

## CÁCH 1: CHẠY DEMO TỰ ĐỘNG (KHUYẾN NGHỊ)

### Bước 1: Cài đặt dependencies
```bash
make setup
```

### Bước 2: Chạy demo tự động
```bash
make demo
```

**Demo này sẽ tự động:**
1. Khởi động server trên port 9000
2. Tạo 3 clients (client1, client2, client3)
3. Client1 chia sẻ file "file1.txt"
4. Client2 chia sẻ file "file2.txt"
5. Client3 tải file từ client1 và client2
6. Kiểm tra tính toàn vẹn dữ liệu (integrity check)
7. Tự động tắt sau khi hoàn thành

**Kết quả mong đợi:**
```
===========================================================
P2P File Sharing System - Demo Topology
===========================================================

[1] Starting server on port 9000...
[2] Starting 3 clients...
[3] Client1 publishes 'sample1.txt'...
[4] Client2 publishes 'sample2.txt'...
[5] Client3 fetches 'file1.txt' from Client1...
[6] Client3 fetches 'file2.txt' from Client2...
[7] Client1 fetches 'file2.txt' from Client2...
[8] Verifying downloads...
  ✓ Client3 received file1.txt
  ✓ Client3 received file2.txt
  ✓ Client1 received file2.txt
  ✓ file1.txt integrity verified
  ✓ file2.txt integrity verified
[9] Testing server admin commands...
[10] Shutting down...

===========================================================
Demo completed successfully!
===========================================================
```

---

## CÁCH 2: CHẠY THỦ CÔNG (CHO THẤY CHI TIẾT)

### Bước 1: Chuẩn bị môi trường
```bash
# Cài đặt dependencies
make setup
```

### Bước 2: Tạo thư mục demo
```bash
# Tạo thư mục cho 2 clients
mkdir -p demo/client_A/shared demo/client_A/downloads
mkdir -p demo/client_B/shared demo/client_B/downloads

# Tạo file mẫu để chia sẻ
echo "Đây là file từ Client A" > demo/client_A/shared/fileA.txt
echo "Đây là file từ Client B" > demo/client_B/shared/fileB.txt
```

### Bước 3: Mở 3 terminal

#### **Terminal 1: Chạy Server**
```bash
make run-server
# Hoặc:
python -m server.server_main --port 9000
```

Kết quả:
```
Server started on 127.0.0.1:9000
Admin shell ready. Commands: discover <hostname>, ping <hostname>, quit
>
```

#### **Terminal 2: Chạy Client A**
```bash
python -m client.client_main \
    --hostname clientA \
    --port 9101 \
    --shared-dir demo/client_A/shared \
    --download-dir demo/client_A/downloads
```

Kết quả:
```
Client 'clientA' started on port 9101
Registered with server
Shared directory: demo/client_A/shared
Download directory: demo/client_A/downloads

Client shell ready. Commands: publish <lname> <fname>, fetch <fname>, quit
>
```

#### **Terminal 3: Chạy Client B**
```bash
python -m client.client_main \
    --hostname clientB \
    --port 9102 \
    --shared-dir demo/client_B/shared \
    --download-dir demo/client_B/downloads
```

Kết quả tương tự Client A.

---

### Bước 4: Demo chia sẻ file

#### **Tại Terminal Client A:**
```
> publish fileA.txt my_file_A.txt
```

Kết quả:
```
Published 'my_file_A.txt' -> 'fileA.txt'
```

#### **Tại Terminal Client B:**
```
> publish fileB.txt my_file_B.txt
```

Kết quả:
```
Published 'my_file_B.txt' -> 'fileB.txt'
```

---

### Bước 5: Demo tải file

#### **Tại Terminal Client A (tải file từ Client B):**
```
> fetch my_file_B.txt
```

Kết quả:
```
Found 1 peer(s) with 'my_file_B.txt'
Selected peer: clientB (RTT: 0.002s)
Downloading my_file_B.txt (30 bytes) from clientB
Download complete: my_file_B.txt (0.50 MB/s)
File saved to: demo/client_A/downloads/my_file_B.txt
```

#### **Tại Terminal Client B (tải file từ Client A):**
```
> fetch my_file_A.txt
```

Kết quả tương tự.

---

### Bước 6: Kiểm tra trên Server

#### **Tại Terminal Server:**

**Xem file của Client A:**
```
> discover clientA
```

Kết quả:
```
Files on clientA:
  - my_file_A.txt
```

**Kiểm tra Client B còn online:**
```
> ping clientB
```

Kết quả:
```
clientB is alive at 127.0.0.1:9102
```

---

### Bước 7: Thoát

**Tại mỗi terminal client:**
```
> quit
```

**Tại terminal server:**
```
> quit
```

---

## CÁCH 3: CHẠY TEST SUITE

Để kiểm tra toàn bộ chức năng:

```bash
make test
```

**Kết quả mong đợi: 20/20 tests PASSED**

```
================================ test session starts =================================
platform linux -- Python 3.11.x
collected 20 items

tests/test_protocol.py .....                                              [ 25%]
tests/test_registry.py .........                                          [ 70%]
tests/test_transfer_small.py ..                                           [ 80%]
tests/test_transfer_large_parallel.py ..                                  [ 90%]
tests/test_integrity_resume.py ..                                         [100%]

================================ 20 passed in 5.43s ==================================
```

---

## GIẢI THÍCH DEMO CHO THẦY

### 1. Kiến trúc hệ thống
```
        ┌──────────────────────────┐
        │  Central Index Server    │
        │  (Port 9000)             │
        │  - Quản lý danh sách     │
        │  - Index file            │
        └──────────┬───────────────┘
                   │
        ┌──────────┼──────────┐
        │          │          │
   ┌────▼───┐ ┌───▼────┐ ┌──▼─────┐
   │Client A│ │Client B│ │Client C│
   │Port    │ │Port    │ │Port    │
   │9101    │ │9102    │ │9103    │
   └────┬───┘ └───┬────┘ └──┬─────┘
        │         │         │
        └─────────┴─────────┘
       Truyền file trực tiếp P2P
```

### 2. Quy trình hoạt động

**Bước 1: Đăng ký**
- Client kết nối đến Server
- Gửi thông tin: hostname, port, danh sách file
- Server lưu vào registry

**Bước 2: Publish file**
- Client thông báo server về file mới
- Server cập nhật file index

**Bước 3: Fetch file**
- Client hỏi server: "Ai có file X?"
- Server trả về danh sách peer có file
- Client ping các peer, chọn peer tốt nhất
- Client kết nối trực tiếp peer, tải file P2P

**Bước 4: Heartbeat**
- Client gửi tín hiệu "còn sống" mỗi 30 giây
- Server cập nhật last_seen
- Nếu > 60s không nhận → client offline

### 3. Tính năng nổi bật

✓ **Truyền file trực tiếp P2P** - Không qua server, tốc độ cao

✓ **Kiểm tra toàn vẹn dữ liệu** - SHA-256 hash verification

✓ **Resume download** - Tiếp tục tải nếu bị gián đoạn

✓ **Peer selection** - Tự động chọn peer nhanh nhất

✓ **Concurrent transfers** - Nhiều upload/download đồng thời

✓ **Registry persistence** - Lưu trạng thái server

---

## TRẢ LỜI CÂU HỎI THẦY CÓ THỂ HỎI

### Q1: Tại sao cần Server trung tâm?
**A:** Server chỉ lưu thông tin "ai có file gì", không lưu file thực tế. Peer tìm nhau qua server, sau đó truyền file trực tiếp P2P.

### Q2: Làm sao đảm bảo file không bị hỏng?
**A:** Dùng SHA-256 hash:
- Mỗi chunk có hash riêng → verify ngay khi nhận
- Toàn bộ file có hash → verify sau khi tải xong
- Nếu sai → xóa file, báo lỗi

### Q3: Resume download hoạt động thế nào?
**A:**
- Lưu offset (số byte đã tải)
- Lần sau gửi GET với offset
- Peer gửi từ vị trí offset
- Client nối tiếp vào file (append mode)

### Q4: Peer selection là gì?
**A:**
- Ping tất cả peer có file
- Đo RTT (Round Trip Time)
- Tính score = 1/(RTT + 0.001)
- Chọn peer có RTT thấp nhất

### Q5: Hệ thống có bảo mật không?
**A:**
- Path traversal protection (không cho truy cập file ngoài shared dir)
- SHA-256 verification (đảm bảo file không bị sửa)
- Timeout mechanisms (tránh DoS)

---

## GỢI Ý DEMO TRỰC QUAN

### Demo ngắn gọn (5 phút):
1. Chạy `make demo` → Xem kết quả tự động
2. Giải thích kiến trúc bằng sơ đồ
3. Chạy `make test` → Xem 20/20 tests pass

### Demo chi tiết (10-15 phút):
1. Mở 3 terminal (Server + 2 Clients)
2. Publish file từ Client A
3. Fetch file từ Client B
4. Dùng lệnh `discover`, `ping` trên server
5. Giải thích flow: REGISTER → PUBLISH → QUERY → P2P transfer
6. Xem file đã tải trong thư mục downloads
7. Kiểm tra integrity bằng SHA-256

---

## LIÊN HỆ

Nếu có vấn đề khi chạy demo, kiểm tra:

1. **Port đã được sử dụng:**
   ```bash
   lsof -i :9000  # Kiểm tra port 9000
   kill -9 <PID>  # Tắt process đang dùng port
   ```

2. **Python version:**
   ```bash
   python --version  # Cần >= 3.11
   ```

3. **Dependencies chưa cài:**
   ```bash
   make setup  # Cài lại
   ```

---

**Chúc bạn demo thành công! 🎉**
