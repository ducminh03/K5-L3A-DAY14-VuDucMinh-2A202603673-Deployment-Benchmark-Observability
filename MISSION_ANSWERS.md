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

#### 1. Base image: `python:3.11`

#### 2. Working directory: `/app`

#### 3. Lý do `COPY requirements.txt` trước khi copy source code: Để cài dep trước, tạo lớp cache => sau này code có thay đổi thì chỉ là lớp dưới, không ảnh hưởng. Tất nhiên khi app mở rộng thì cả lớp dep này cũng có thể bị thay đổi do `requirements.txt` thay đổi.

#### 4. Sự khác nhau giữa `CMD` và `ENTRYPOINT`

- `CMD`: lệnh hoặc tham số mặc định khi container khởi động; có thể bị ghi đè hoàn toàn bằng lệnh đặt ở cuối `docker run`.
- `ENTRYPOINT`: chương trình thực thi chính của container; thường vẫn được giữ nguyên, còn tham số ở cuối `docker run` sẽ được truyền thêm vào chương trình đó.
- Có thể kết hợp cả hai: `ENTRYPOINT` định nghĩa executable, `CMD` cung cấp tham số mặc định.

Ví dụ:

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--port", "8000"]
```

Trong ví dụ này, `CMD` có thể thay đổi bằng `docker run image --port 9000`, còn `ENTRYPOINT` vẫn chạy `python app.py`.

### Exercise 2.2: Build và run

#### Lệnh build image

```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
```

**Kết quả build:**

```
[+] Building 656.4s (12/12) FINISHED                                                                     docker:desktop-linux
 => [internal] load build definition from Dockerfile                                                                     0.1s
 => => transferring dockerfile: 1.36kB                                                                                   0.0s
 => [internal] load metadata for docker.io/library/python:3.11                                                           5.8s
 => [internal] load .dockerignore                                                                                        0.0s
 => => transferring context: 486B                                                                                        0.0s
 => [1/7] FROM docker.io/library/python:3.11@sha256:1829edfddfa93b381f575e89221003055b9a0b6cf03dfe786edd798bb6817e83   517.8s
 => => resolve docker.io/library/python:3.11@sha256:1829edfddfa93b381f575e89221003055b9a0b6cf03dfe786edd798bb6817e83     0.1s
 => => sha256:333ec3b0a17780f7f0d76ec480f87c3e90842b8f38fe7dc31c33dc9b2a571ea8 6.09MB / 6.09MB                           5.8s
 => => sha256:807982f6af4fc03f2d394347e6e9bcaab579df144f168742316271ff8f226ac1 237.38MB / 237.38MB                     507.1s
 => => sha256:fa3fd96fbefb9f21799967d25979a970d25083808eb3abcfc9bc4511b501fa58 24.08MB / 24.08MB                        54.3s
 => => sha256:c2acb1a7e2440113768de2c5a81ca8c6c89281a433d4e9f05eecc3082aac2c04 67.82MB / 67.82MB                       323.7s
 => => sha256:405896332490b36beec1da0a0cbe7bc4223672fb5555f52907f26318e8826e32 247B / 247B                               0.4s
 => => sha256:f0be6780dc18eb58a60fabb4c8bedb0c3ea852f4f3c7d199d0dd2ca073068d3c 25.65MB / 25.65MB                        19.5s
 => => sha256:133e21d3e00f3d2179ee10bc16a86f143abd8d09d42089b8b8542a3f01c80448 49.41MB / 49.41MB                        36.7s
 => => extracting sha256:133e21d3e00f3d2179ee10bc16a86f143abd8d09d42089b8b8542a3f01c80448                                2.7s
 => => extracting sha256:f0be6780dc18eb58a60fabb4c8bedb0c3ea852f4f3c7d199d0dd2ca073068d3c                                1.0s
 => => extracting sha256:c2acb1a7e2440113768de2c5a81ca8c6c89281a433d4e9f05eecc3082aac2c04                                3.0s
 => => extracting sha256:807982f6af4fc03f2d394347e6e9bcaab579df144f168742316271ff8f226ac1                                8.9s
 => => extracting sha256:333ec3b0a17780f7f0d76ec480f87c3e90842b8f38fe7dc31c33dc9b2a571ea8                                0.3s
 => => extracting sha256:fa3fd96fbefb9f21799967d25979a970d25083808eb3abcfc9bc4511b501fa58                                1.0s
 => => extracting sha256:405896332490b36beec1da0a0cbe7bc4223672fb5555f52907f26318e8826e32                                0.0s
 => [internal] load build context                                                                                        0.1s
 => => transferring context: 2.61kB                                                                                      0.0s
 => [2/7] WORKDIR /app                                                                                                   0.5s
 => [3/7] COPY 02-docker/develop/requirements.txt .                                                                      0.1s
 => [4/7] RUN pip install --no-cache-dir -r requirements.txt                                                           126.8s
 => [5/7] COPY 02-docker/develop/app.py .                                                                                0.1s
 => [6/7] RUN mkdir -p utils                                                                                             0.4s
 => [7/7] COPY utils/mock_llm.py utils/                                                                                  0.1s
 => exporting to image                                                                                                   4.3s
 => => exporting layers                                                                                                  3.1s
 => => exporting manifest sha256:fec649e1c065657c9f8b8e1d74774af78a1a0227051ae1c008899cb10981ee5e                        0.0s
 => => exporting config sha256:3908ebd24f5968b9920cfd7af0100bd3e64e135f8fed178fb5f394063417b489                          0.0s
 => => exporting attestation manifest sha256:62adcdbf26e911f13ed31ae6092c94300ae1a3344c7e15a00c2028d75bbce15d            0.0s
 => => exporting manifest list sha256:0e22f275ecc4b19ba31680485a5e3bf075152d3e5f9de611c730c68209de6b66                   0.0s
 => => naming to docker.io/library/agent-develop:latest                                                                  0.0s
 => => unpacking to docker.io/library/agent-develop:latest                                                               0.9s

View build details: docker-desktop://dashboard/build/desktop-linux/desktop-linux/8pqflf8ogzmqs3ud7vkizjtuh
```

#### Lệnh chạy container

```bash
docker run -p 8000:8000 my-agent:develop
```

**Kết quả chạy container:**

```
INFO:     Started server process [1]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

#### Lệnh test endpoint

```powershell
iwr -Method POST -Uri "http://localhost:8000/ask?question=What%20is%20Docker?" | Select-Object -ExpandProperty Content
```

**Kết quả response:** `{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}`

#### Kiểm tra kích thước image

```bash
docker images my-agent:develop
```

**Kích thước image:** 1,67GB

```
                                                                                                          i Info →   U  In Use
IMAGE                  ID             DISK USAGE   CONTENT SIZE   EXTRA
agent-develop:latest   0e22f275ecc4       1.67GB          426MB    U   
```

### Exercise 2.3: Multi-stage build

#### Stage 1: Builder

**Stage 1 làm gì?**: Tải, compile và cài đặt dependency của ứng dụng, cache thành 1 layer

#### Stage 2: Runtime

**Stage 2 làm gì?**: build contaier thành phẩm, chỉ bao gồm các lib cần thiết để chạy

#### Lý do image production nhỏ hơn

**Trả lời:** Image production nhỏ hơn vì sử dụng base image `python:3.11-slim`, trong khi bản develop sử dụng image `python:3.11` đầy đủ. Bản `slim` loại bỏ nhiều package và công cụ không cần thiết khi chạy ứng dụng, nên giảm đáng kể kích thước image. Ngoài ra, Dockerfile production dùng multi-stage build và chỉ copy các dependency cần thiết sang stage runtime.

**Nguồn:** [Dockerfile develop](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/02-docker/develop/Dockerfile) và [Dockerfile production](E:/vinuni/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability/02-docker/production/Dockerfile)

#### Kết quả build và so sánh image

```bash
docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .
docker images my-agent:advanced
```

**Lưu ý về hướng dẫn đề bài:** Đề bài yêu cầu `cd ../production` rồi chạy lệnh, nhưng cách này làm build context chỉ còn thư mục `02-docker/production`. Khi đó Dockerfile không tìm thấy `utils/mock_llm.py` ở root repo. Vì vậy, lệnh được thực hiện tại **gốc repo**, chỉ định Dockerfile bằng `-f 02-docker/production/Dockerfile`.

**Kết quả lệnh thực tế:** Build thành công.

```text
REPOSITORY   TAG        SIZE
my-agent     advanced   240MB
```

| Image | Kích thước |
|-------|------------|
| `my-agent:develop` | 1,67 GB |
| `my-agent:advanced` | 240 MB |

**Chênh lệch:** 1,43 GB (-85,6%) so với image develop.

### Exercise 2.4: Docker Compose stack

#### Architecture diagram

```mermaid
flowchart LR
    Client[Client / Browser / curl]
    Nginx[Nginx\nReverse proxy + Load balancer\nPorts 80 / 443]
    Agent[Agent service\nFastAPI application\nPORT 8000]
    Redis[Redis\nSession cache + rate limiting\nPort 6379]
    Qdrant[Qdrant\nVector database / RAG\nPort 6333]
    RedisVol[(redis_data volume)]
    QdrantVol[(qdrant_data volume)]

    Client -->|HTTP / HTTPS| Nginx
    Nginx -->|Internal network| Agent
    Agent -->|REDIS_URL| Redis
    Agent -->|QDRANT_URL| Qdrant
    Redis --> RedisVol
    Qdrant --> QdrantVol
```

**Network:** Các service giao tiếp qua Docker network nội bộ `internal`. Chỉ Nginx expose cổng ra ngoài; Agent, Redis và Qdrant không expose trực tiếp ra host.

#### Các service được khởi động

**Danh sách service:**

- `nginx`: reverse proxy/load balancer, expose cổng `80` và `443`.
- `agent`: FastAPI AI agent, chạy nội bộ trên cổng `8000`.
- `redis`: session cache và rate limiting, cổng nội bộ `6379`.
- `qdrant`: vector database cho RAG, cổng nội bộ `6333`.

#### Cách các service communicate với nhau

**Mô tả:** Client gửi request tới Nginx qua cổng `80/443`. Nginx chuyển request qua Docker network `internal` tới Agent. Agent kết nối tới Redis bằng `REDIS_URL=redis://redis:6379/0` và tới Qdrant bằng `QDRANT_URL=http://qdrant:6333`. Redis và Qdrant lưu dữ liệu vào các volume tương ứng.

#### Lệnh khởi động stack

```bash
cd 02-docker/production
docker compose -p day14-ex2-4 up -d --build
```

**Workdir đúng:** `02-docker/production`. Compose file dùng build context `../..`, Dockerfile `02-docker/production/Dockerfile`, và các volume path tương đối theo thư mục production.

**Lưu ý về đề bài:** Không nên chạy từ workdir khác hoặc `cd` vào thư mục làm mất build context phù hợp. Với stack này, chạy từ `02-docker/production` là đúng vì `docker-compose.yml` nằm tại đây và đã cấu hình context `../..`. File `.env.local` cũng cần tồn tại trong thư mục này.

**Kết quả khởi động:** 4 container được tạo và chạy thành công. `agent`, `redis`, `qdrant` healthy; `nginx` đang chạy và expose `0.0.0.0:80->80` và `0.0.0.0:443->443`.

#### Test health check

```bash
curl http://localhost/health
```

**Response thực tế:**

```text
StatusCode=200
{"status":"ok","uptime_seconds":7.5,"version":"2.0.0","timestamp":"2026-11-25T09:14:29.789293"}
```

#### Test agent endpoint

```bash
curl http://localhost/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
```

**Response thực tế:**

```text
StatusCode=200
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

#### Kết luận Exercise 2.4

**Nhận xét:** Docker Compose đã khởi động đầy đủ stack gồm Nginx, Agent, Redis và Qdrant. Request đi qua Nginx tới Agent thành công; `/health` và `/ask` đều trả HTTP `200 OK`.

## Part 3: Cloud Deployment

### Exercise 3.1: Deploy Railway

#### Thư mục thực hiện

```powershell
cd 03-cloud-deployment/railway
```

#### Cài đặt Railway CLI trên Windows PowerShell

Lệnh `npm i -g @railway/cli` bị chặn install script bởi cơ chế `allowScripts` của npm. Lệnh đúng đã sử dụng là:

```powershell
npm.cmd install -g @railway/cli --allow-scripts=@railway/cli
railway.cmd --version
```

**Kết quả:** `railway 5.75.0`

#### Đăng nhập và khởi tạo project

```powershell
railway.cmd login
railway.cmd init
```

**Project:** `courageous-victory`  
**Project ID:** `6ae38982-a929-4a95-9087-31877b95bad9`  
**Railway dashboard:** https://railway.com/project/6ae38982-a929-4a95-9087-31877b95bad9

#### Cấu hình biến môi trường

```powershell
railway.cmd variables set PORT=8000 AGENT_API_KEY=my-secret-key ENVIRONMENT=production
```

Biến `AGENT_API_KEY` đã được cấu hình trên Railway; giá trị secret không ghi vào báo cáo.

#### Lỗi deployment ban đầu và cách sửa

Deployment đầu tiên bị crash với log:

```text
ERROR:    Error loading ASGI app. Could not import module "main".
```

Railway/Nixpacks tự chọn lệnh `uvicorn main:app`, trong khi source code dùng file `app.py`. Đã thêm [Procfile](./03-cloud-deployment/railway/Procfile) để chỉ rõ lệnh khởi động:

```text
web: uvicorn app:app --host 0.0.0.0 --port $PORT
```

Sau đó chạy deploy lại:

```powershell
railway.cmd up --detach
```

#### Public URL

https://courageous-victory-production-dfc21644.up.railway.app

![alt text](image.png)

**Trạng thái:** Railway service `Online`  
**Deployment ID:** `40607b46-fecd-4d14-a6d6-e9e29973a333`

#### Test health check

```powershell
iwr -UseBasicParsing https://courageous-victory-production-dfc21644.up.railway.app/health
```

**Kết quả thực tế:**

```text
StatusCode=200
{"status":"ok","uptime_seconds":41.5,"platform":"Railway","timestamp":"2026-11-25T09:29:44.765033+00:00"}
```

#### Test agent endpoint

```powershell
iwr -UseBasicParsing -Method POST `
  -Uri "https://courageous-victory-production-dfc21644.up.railway.app/ask" `
  -ContentType "application/json" `
  -Body '{"question":"What is Docker?"}'
```

**Kết quả thực tế:**

```text
StatusCode=200
{"question":"What is Docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","platform":"Railway"}
```

#### Kết luận

Ứng dụng đã được deploy thành công lên Railway. Public URL hoạt động, endpoint `/health` trả `200 OK`, và endpoint `/ask` trả response JSON với `200 OK`.

### Exercise 3.2: Deploy Render

#### Cấu hình và workdir

Repository URL: https://github.com/ducminh03/K5-L3A-DAY14-VuDucMinh-2A202603673-Deployment-Benchmark-Observability  
Blueprint path: `03-cloud-deployment/render/render.yaml`  
Source root directory: `03-cloud-deployment/railway`

Render Blueprint ban đầu không deploy trực tiếp từ thư mục chứa `render.yaml` vì source code (`app.py`, `requirements.txt`) nằm trong thư mục `03-cloud-deployment/railway`. Đã thêm `rootDir: 03-cloud-deployment/railway` vào web service để Render dùng đúng source.

Redis Blueprint cũng yêu cầu IP allow list, nên đã thêm allow list cho lab testing trong [render.yaml](./03-cloud-deployment/render/render.yaml). Thay đổi đã được push lên GitHub ở commit `e5a6bcd`.

#### So sánh `render.yaml` và `railway.toml`

| Nội dung | Render | Railway |
|----------|--------|---------|
| Định dạng | YAML Blueprint | TOML Config as Code |
| Build | `runtime: python`, `buildCommand: pip install -r requirements.txt` | Railway tự detect Nixpacks từ `builder = "NIXPACKS"` |
| Start command | `uvicorn app:app --host 0.0.0.0 --port $PORT` | `uvicorn app:app --host 0.0.0.0 --port $PORT` |
| Health check | `healthCheckPath: /health` | `healthcheckPath = "/health"` |
| Region | Khai báo `singapore` | Railway tự chọn/được cấu hình theo project |
| Environment variables | Khai báo trong `envVars`, secret dùng `sync: false` hoặc `generateValue` | Đặt bằng `railway variables set` hoặc Dashboard |
| Redis | Khai báo service Redis `agent-cache` trong cùng Blueprint | Không khai báo Redis trong `railway.toml` của bài này |
| Auto deploy | `autoDeploy: true` khi kết nối GitHub | Có thể deploy bằng CLI hoặc kết nối source |

#### Kết quả deploy

Render Blueprint `day14-deployment-benchmark` đã tạo thành công:

- Redis service: `agent-cache`
- Web service: `ai-agent`
- Plan: Free
- Public URL: https://ai-agent-n2yy.onrender.com

![alt text](image-1.png)

#### Test health check

```powershell
iwr -UseBasicParsing https://ai-agent-n2yy.onrender.com/health
```

**Kết quả thực tế:**

```text
StatusCode=200
{"status":"ok","uptime_seconds":43.4,"platform":"Railway","timestamp":"2026-11-25T09:40:16.293179+00:00"}
```

#### Test agent endpoint

```powershell
iwr -UseBasicParsing -Method POST `
  -Uri "https://ai-agent-n2yy.onrender.com/ask" `
  -ContentType "application/json" `
  -Body '{"question":"What is Docker?"}'
```

**Kết quả thực tế:**

```text
StatusCode=200
{"question":"What is Docker?","answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!","platform":"Railway"}
```

#### Kết luận

Ứng dụng đã deploy thành công trên Render. Blueprint tạo được cả Redis và web service; public URL trả HTTP `200 OK` cho cả `/health` và `/ask`. Vì Render dùng source chung từ `03-cloud-deployment/railway`, response vẫn ghi `platform: Railway` theo code mẫu, nhưng service thực tế đang chạy trên Render.

### Exercise 3.3: GCP Cloud Run (Optional)

## Part 4: API Security

### Exercise 4.1: API Key authentication

#### API key được check ở đâu?

API key được lấy từ biến môi trường `AGENT_API_KEY` và đọc tại phần cấu hình API key trong [app.py](./04-api-gateway/develop/app.py). Dependency `verify_api_key()` kiểm tra header `X-API-Key` và được inject vào endpoint `POST /ask` bằng `Depends(verify_api_key)`.

#### Điều gì xảy ra nếu thiếu hoặc sai key?

- Không có header `X-API-Key`: trả HTTP `401 Unauthorized` với thông báo yêu cầu API key.
- Có API key nhưng sai: trả HTTP `403 Forbidden` với thông báo `Invalid API key.`.
- API key đúng: request được xử lý và trả HTTP `200 OK`.

#### Cách rotate key

Đổi giá trị biến môi trường `AGENT_API_KEY` sang key mới trong môi trường deploy, sau đó restart/redeploy service. Không hardcode key trong source code và không commit key thật vào Git.

#### Cấu hình và chạy local

```powershell
$env:AGENT_API_KEY='secret-key-123'
python app.py
```

#### Test không có key

```powershell
iwr -UseBasicParsing -SkipHttpErrorCheck -Method POST `
  -Uri "http://localhost:8000/ask" `
  -ContentType "application/json" `
  -Body '{"question":"Hello"}'
```

```text
StatusCode=401
{"detail":"Missing API key. Include header: X-API-Key: <your-key>"}
```

#### Test với key sai

```powershell
iwr -UseBasicParsing -SkipHttpErrorCheck -Method POST `
  -Uri "http://localhost:8000/ask" `
  -Headers @{"X-API-Key"="wrong-key"} `
  -ContentType "application/json" `
  -Body '{"question":"Hello"}'
```

```text
StatusCode=403
{"detail":"Invalid API key."}
```

#### Test với key đúng

```powershell
iwr -UseBasicParsing -Method POST `
  -Uri "http://localhost:8000/ask" `
  -Headers @{"X-API-Key"="secret-key-123"} `
  -ContentType "application/json" `
  -Body '{"question":"Hello"}'
```

```text
StatusCode=200
{"question":"Hello","answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
```

### Exercise 4.2: JWT authentication

#### JWT flow

1. Client gửi username/password tới `POST /auth/token`.
2. Server xác thực thông tin đăng nhập trong `auth.py`.
3. Server tạo JWT chứa `sub`, `role`, `iat` và `exp` bằng thuật toán HS256.
4. Client gửi token trong header `Authorization: Bearer <token>` khi gọi endpoint được bảo vệ.
5. Server verify chữ ký và hạn sử dụng token trước khi xử lý request.

#### Lưu ý về hướng dẫn đề bài

Đề bài dùng endpoint `/token` với tài khoản `admin/secret`, nhưng code production thực tế dùng endpoint `/auth/token` và demo credentials `student/demo123` hoặc `teacher/teach456`. Vì vậy, test được thực hiện theo code thực tế.

Port `8000` đang được dùng bởi Exercise 4.1, nên JWT app được chạy ở port `8001` để tránh xung đột.

#### Cài dependency và chạy app

```powershell
cd 04-api-gateway/production
python -m pip install -r requirements.txt
$env:JWT_SECRET='exercise-4-2-secret'
$env:PORT='8001'
python app.py
```

Dependency `PyJWT` được cài bổ sung vì lần chạy đầu báo `ModuleNotFoundError: No module named 'jwt'`.

#### Lấy JWT token

```powershell
$tokenResponse = iwr -UseBasicParsing -Method POST `
  -Uri "http://localhost:8001/auth/token" `
  -ContentType "application/json" `
  -Body '{"username":"student","password":"demo123"}'
```

**Kết quả:** HTTP `200 OK`, token nhận được, thời hạn `60` phút. Token thật không dán vào báo cáo.

#### Test endpoint sai theo đề bài

```powershell
iwr -UseBasicParsing -SkipHttpErrorCheck -Method POST `
  -Uri "http://localhost:8001/token" `
  -ContentType "application/json" `
  -Body '{"username":"admin","password":"secret"}'
```

```text
StatusCode=404
{"detail":"Not Found"}
```

#### Test không có token

```powershell
iwr -UseBasicParsing -SkipHttpErrorCheck -Method POST `
  -Uri "http://localhost:8001/ask" `
  -ContentType "application/json" `
  -Body '{"question":"Explain JWT"}'
```

```text
StatusCode=401
{"detail":"Authentication required. Include: Authorization: Bearer <token>"}
```

#### Test token sai

```powershell
iwr -UseBasicParsing -SkipHttpErrorCheck -Method POST `
  -Uri "http://localhost:8001/ask" `
  -Headers @{Authorization="Bearer invalid-token"} `
  -ContentType "application/json" `
  -Body '{"question":"Explain JWT"}'
```

```text
StatusCode=403
{"detail":"Invalid token."}
```

#### Test token hợp lệ

```powershell
$token = ($tokenResponse.Content | ConvertFrom-Json).access_token
iwr -UseBasicParsing -Method POST `
  -Uri "http://localhost:8001/ask" `
  -Headers @{Authorization="Bearer $token"} `
  -ContentType "application/json" `
  -Body '{"question":"Explain JWT"}'
```

```text
StatusCode=200
{"question":"Explain JWT","answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.","usage":{"requests_remaining":9,"budget_remaining_usd":1.9e-05}}
```

### Exercise 4.3: Rate limiting

#### Thuật toán

Code sử dụng **Sliding Window Counter**. Mỗi user có một `deque` chứa timestamp của các request. Trước mỗi request, các timestamp cũ hơn `60` giây sẽ bị xóa; nếu số request còn lại đạt giới hạn thì server trả HTTP `429 Too Many Requests`.

#### Giới hạn request

- User: `10 requests / 60 giây`.
- Admin: `100 requests / 60 giây`.

Trong mô tả có chỗ ghi “60 phút”, nhưng code thực tế đặt `window_seconds=60`, tức là **60 giây**.

#### Cách admin vượt giới hạn user

Admin không bypass thuật toán bằng cách phá cơ chế rate limit. Khi JWT có `role=admin`, code chọn limiter riêng:

```python
limiter = rate_limiter_admin if role == "admin" else rate_limiter_user
```

Vì vậy, cách hợp lệ để có limit cao hơn là đăng nhập bằng tài khoản có role `admin` (`teacher/teach456`). Admin được cấp limit `100 requests / 60 giây`, nên 20 request liên tiếp không bị chặn. Nếu admin vượt quá 100 request trong cùng cửa sổ 60 giây thì vẫn nhận HTTP `429`.

#### Test PowerShell — user

```powershell
$base='http://localhost:8001'
$student = (iwr -UseBasicParsing -Method POST -Uri "$base/auth/token" -ContentType 'application/json' -Body '{"username":"student","password":"demo123"}').Content | ConvertFrom-Json | Select-Object -ExpandProperty access_token
$body='{ "question": "Rate limit test" }'
1..20 | % {
  $resp=iwr -UseBasicParsing -SkipHttpErrorCheck -Method POST -Uri "$base/ask" -Headers @{Authorization="Bearer $student"} -ContentType 'application/json' -Body $body
  "{0}:{1}" -f $_,$resp.StatusCode
}
```

**Kết quả:** Request `1–10` trả `200`; request `11–20` trả `429`.

```text
USER_200=10 USER_429=10
```

#### Test PowerShell — admin

```powershell
$teacher = (iwr -UseBasicParsing -Method POST -Uri "$base/auth/token" -ContentType 'application/json' -Body '{"username":"teacher","password":"teach456"}').Content | ConvertFrom-Json | Select-Object -ExpandProperty access_token
1..20 | % {
  $resp=iwr -UseBasicParsing -SkipHttpErrorCheck -Method POST -Uri "$base/ask" -Headers @{Authorization="Bearer $teacher"} -ContentType 'application/json' -Body $body
  "{0}:{1}" -f $_,$resp.StatusCode
}
```

**Kết quả:** Cả 20 request của admin đều trả `200`.

```text
ADMIN_200=20 ADMIN_429=0
```

#### Kết luận

Rate limiter hoạt động đúng theo role: user bị giới hạn sau 10 request/60 giây, còn admin có ngưỡng 100 request/60 giây. Đây là phân tầng quyền hợp lệ, không phải bypass trái phép.

### Exercise 4.4: Cost guard
Lab đề bài đã implement sẵn.

## Part 5: Scaling & Reliability

### Exercise 5.1: Health checks
Lab đề bài đã implement sẵn.
### Exercise 5.2: Graceful shutdown
Lab đề bài đã implement sẵn. Không test được vì Windows không có cơ chế signal giống các hệ điều hành POSIX.
### Exercise 5.3: Stateless design
Lab đề bài đã implement sẵn.
### Exercise 5.4: Load balancing

#### Lệnh khởi động stack

Chạy từ thư mục `05-scaling-reliability/production` với project name riêng:

```powershell
docker compose -p k5-day14-scaling up -d --build --scale agent=3
```

#### Trạng thái service

Stack khởi động thành công với:

- `agent-1`: healthy
- `agent-2`: healthy
- `agent-3`: healthy
- `redis`: healthy
- `nginx`: chạy và expose `localhost:8080`

#### Test 10 requests

```powershell
$body='{ "question": "Request TEST", "session_id": "load-test-5-4" }'
1..10 | % {
  $resp=iwr -UseBasicParsing -Method POST -Uri 'http://localhost:8080/chat' -ContentType 'application/json' -Body ($body -replace 'TEST',$_)
  $j=$resp.Content | ConvertFrom-Json
  "{0}: served_by={1} status={2}" -f $_,$j.served_by,$resp.StatusCode
}
```

**Kết quả thực tế:**

```text
1: served_by=instance-4429f9 status=200
2: served_by=instance-1ce1aa status=200
3: served_by=instance-aedfd7 status=200
4: served_by=instance-4429f9 status=200
5: served_by=instance-1ce1aa status=200
6: served_by=instance-aedfd7 status=200
7: served_by=instance-4429f9 status=200
8: served_by=instance-1ce1aa status=200
9: served_by=instance-aedfd7 status=200
10: served_by=instance-4429f9 status=200
```

**Kết luận:** Nginx đã phân phối request qua cả 3 agent instance; tất cả request trả HTTP `200`.

### Exercise 5.5: Test stateless

#### Lệnh chạy test

```powershell
$env:PYTHONIOENCODING='utf-8'
python test_stateless.py
```

Lần chạy đầu trên PowerShell bị `UnicodeEncodeError` do console dùng CP1252. Chạy lại với `PYTHONIOENCODING=utf-8` thì test hoàn tất.

#### Kết quả thực tế

```text
Session ID: 1de12289-69f2-4627-a3b8-0f15a54b91b2
Total requests: 5
Instances used: {'instance-4429f9', 'instance-aedfd7', 'instance-1ce1aa'}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
✅ Session history preserved across all instances via Redis!
```

#### Kiểm tra failover

Sau khi dừng `agent-1`, request tiếp theo vẫn được xử lý bởi instance còn lại:

```text
StatusCode=200
served_by=instance-aedfd7
storage=redis
turn=7
```

Traffic tiếp tục được phục vụ và conversation vẫn đọc được từ Redis. Lưu ý: file `test_stateless.py` hiện tại không tự kill random instance như phần mô tả; việc dừng instance và kiểm tra failover đã được thực hiện bổ sung thủ công.

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

