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

