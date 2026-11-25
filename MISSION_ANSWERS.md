# Day 14 Lab - Mission Answers

## Thông tin sinh viên

- Họ và tên: Vũ Đức Minh
- MSSV: 2A202603673
- Lớp: 3A

## Thiết đặt máy thực hiện lab

- Hệ điều hành: Windows 10 Home Single Language, version 26H2, build 26300.9974
- Python: `3.14.4`
- Windows PowerShell 5.1: `5.1.26100.9818`
- PowerShell 7+: `7.7.0`
- Docker: `29.4.0`, build `9d7ad9f`
- Docker Compose: `v5.1.1`

## Part 1: Localhost vs Production

### Exercise 1.1: Phát hiện anti-patterns

#### 1. Hardcoded secrets trong source code — nguồn: [app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)

```python
# ❌ Vấn đề 1: API key hardcode trong code
# Nếu push lên GitHub → key bị lộ ngay lập tức
OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"
DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"
```

Không thay đổi được cấu hình khi môi trường bên ngoài thay đổi. Đồng thời, với những API key dạng pay-as-you-go, lộ key sẽ khiến tin tặc sử dụng dịch vụ của mình đăng ký, gây thiệt hại về chi phí.

#### 2. Cấu hình cố định, không có config management — nguồn: [app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)

```python
# ❌ Vấn đề 2: Không có config management
DEBUG = True
MAX_TOKENS = 500
```

Tương tự như trên, không tùy chỉnh được các tham số khi môi trường bên ngoài hoặc yêu cầu thay đổi

#### 3. Dùng `print()` và log lộ secret — nguồn: [app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)

```python
# ❌ Vấn đề 3: Print thay vì proper logging
print(f"[DEBUG] Got question: {question}")
print(f"[DEBUG] Using key: {OPENAI_API_KEY}")  # ❌ log ra secret!
print(f"[DEBUG] Response: {response}")
```

Log vẫn nằm trên chính server đó => Khi hệ thống phát triển thành nhiều phân hệ, lấy log về để phân tích tốn nhiều thời gian do quá phân tán. Người nào (hoặc kẻ xâm nhập) có quyền truy cập có thể đọc log, bao gồm cả thông tin nhạy cảm.
Log không có cấu trúc => tìm khó

#### 4. Không có health check endpoint — nguồn: [app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)

```python
# ❌ Vấn đề 4: Không có health check endpoint
# Nếu agent crash, platform không biết để restart
```

Hmm, thực ra thì cứ theo dõi xem process có còn hoạt động không là oke mà? Nhưng nó không kiểm tra được người dùng có thực sự nhận phản hồi khi gửi yêu cầu vào web server không.

#### 5. Bind localhost, hardcode port và bật reload — nguồn: [app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)

```python
# ❌ Vấn đề 5: Port cố định — không đọc từ environment
# Trên Railway/Render, PORT được inject qua env var
if __name__ == "__main__":
    print("Starting agent on localhost:8000...")
    uvicorn.run(
        "app:app",
        host="localhost",   # ❌ chỉ chạy được trên local
        port=8000,          # ❌ cứng port
        reload=True         # ❌ debug reload trong production
    )
```

Không xử lý được trường hợp:
- Muốn publish cho tất cả các interface mạng
- Port bị trùng với các app khác

### Exercise 1.2: Chạy basic version

Lệnh `python app.py` đã khởi động server thành công tại `http://localhost:8000`.

```text
Starting agent on localhost:8000...
INFO:     Will watch for changes in these directories: ['E:\\vinuni\\K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability\\01-localhost-vs-production\\develop']
INFO:     Uvicorn running on http://localhost:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [22320] using WatchFiles
INFO:     Started server process [2004]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
```

Tuy nhiên, lệnh kiểm tra API theo hướng dẫn dùng `curl` không chạy trong Windows PowerShell do `curl` có thể được ánh xạ thành alias của `Invoke-WebRequest`, nên cú pháp `curl` dạng Unix không tương thích trực tiếp.

Trên PowerShell có thể dùng `Invoke-RestMethod` hoặc gọi rõ `curl.exe` để gửi request đến endpoint `/ask`.

Đã thực hiện test bằng `Invoke-WebRequest` (`iwr`) trên PowerShell với lệnh:

```powershell
iwr -Method POST -Uri "http://localhost:8000/ask?question=Hello"
```

Để chỉ lấy nội dung JSON của response:

```powershell
(iwr -Method POST -Uri "http://localhost:8000/ask?question=Hello").Content
```

Kết quả kiểm tra thực tế:

```text
StatusCode        : 200
StatusDescription : OK
Content           : {"answer":"Agent đang hoạt động tốt! (mock response) Hỏi thêm câu hỏi đi nhé."}
Server            : uvicorn
Content-Type      : application/json
```

Kết luận: server chạy thành công trên `http://localhost:8000`. Endpoint `POST /ask` nhận request và trả về HTTP `200 OK` cùng response JSON hợp lệ.

Song, vì các lẽ trên, nó không production-ready.

### Exercise 1.3: So sánh với advanced version

| Feature | Develop/Basic | Production/Advanced | Tại sao quan trọng? |
|---------|---------------|---------------------|---------------------|
| Config | Hardcode `OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG`, `MAX_TOKENS` trong source ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)) | Đọc cấu hình từ environment thông qua `Settings` ([config.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/production/config.py)) | Tách cấu hình khỏi mã nguồn, tránh lộ secret và cho phép dùng cùng code ở nhiều môi trường. |
| Health check | Không có endpoint `/health` ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)) | Có `/health` ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/production/app.py)) và `/ready` ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/production/app.py)) | Nền tảng cloud/load balancer biết service còn sống và đã sẵn sàng nhận traffic. |
| Logging | Dùng `print()` và log cả API key ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)) | Dùng `logging` với format JSON và không log secret ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/production/app.py)) | Log có cấu trúc dễ tìm kiếm/phân tích và giảm nguy cơ lộ thông tin nhạy cảm. |
| Shutdown | Bản develop vẫn xử lý graceful shutdown nhờ Uvicorn khi chạy local và nhận `Ctrl+C`, **dù không có lifecycle/cleanup logic riêng trong code, trái với giả định ban đầu của đề bài** | Có lifecycle `lifespan` ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/production/app.py)) và bắt `SIGTERM` ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/production/app.py));  | Graceful shutdown giúp hoàn tất request đang xử lý và đóng service an toàn khi dừng hoặc triển khai lại. |
| Binding và port | Bind `localhost`, port `8000` cố định, `reload=True` ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/develop/app.py)) | Bind theo `settings.host`, đọc port từ config/env và chỉ reload khi debug ([app.py](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/01-localhost-vs-production/production/app.py)) | Cho phép container nhận kết nối từ bên ngoài và tương thích với port do Railway/Render inject. |

## Part 2: Docker Containerization

### Exercise 2.1: Dockerfile cơ bản

### Exercise 2.2: Build và run

### Exercise 2.3: Multi-stage build

### Exercise 2.4: Docker Compose stack

## Part 3: Cloud Deployment

### Exercise 3.1: Deploy Railway

### Exercise 3.2: Deploy Render

### Exercise 3.3: GCP Cloud Run (Optional)

## Part 4: API Security

### Exercise 4.1: API Key authentication

### Exercise 4.2: JWT authentication

### Exercise 4.3: Rate limiting

### Exercise 4.4: Cost guard

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks

### Exercise 5.2: Graceful shutdown

### Exercise 5.3: Stateless design

### Exercise 5.4: Load balancing

### Exercise 5.5: Test stateless

## Part 6: Final Project

### Kết quả triển khai các yêu cầu

### Kết quả `check_production_ready.py`

### Checklist Deliverable từ output script

### Ghi chú implementation

## Part 7: Benchmark & Observability

### Exercise 7.1: Đọc metrics của agent

### Exercise 7.2: Load test & SLO

### Exercise 7.3: Eval & quality gate

### Exercise 7.4: Observability stack

### Exercise 7.5: Tracing

### Exercise 7.6: Alerts

### Exercise 7.7: Benchmark report

## AI Usage Log

### Công cụ hoặc AI đã sử dụng

### Nội dung đã tham khảo

### Phần tự kiểm tra và chỉnh sửa

