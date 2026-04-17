# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

#  Code Lab: Deploy Your AI Agent to Production

> **AICB-P1 · VinUniversity 2026**  
> Thời gian: 3-4 giờ | Độ khó: Intermediate

##  Mục Tiêu

Sau khi hoàn thành lab này, bạn sẽ:
- Hiểu sự khác biệt giữa development và production
- Containerize một AI agent với Docker
- Deploy agent lên cloud platform
- Bảo mật API với authentication và rate limiting
- Thiết kế hệ thống có khả năng scale và reliable

---

##  Yêu Cầu

```bash
 Python 3.11+
 Docker & Docker Compose
 Git
 Text editor (VS Code khuyến nghị)
 Terminal/Command line
```

**Không cần:**
-  OpenAI API key (dùng mock LLM)
-  Credit card
-  Kinh nghiệm DevOps trước đó

---

##  Lộ Trình Lab

| Phần | Thời gian | Nội dung |
|------|-----------|----------|
| **Part 1** | 30 phút | Localhost vs Production |
| **Part 2** | 45 phút | Docker Containerization |
| **Part 3** | 45 phút | Cloud Deployment |
| **Part 4** | 40 phút | API Security |
| **Part 5** | 40 phút | Scaling & Reliability |
| **Part 6** | 60 phút | Final Project |

---

## Part 1: Localhost vs Production (30 phút)

###  Concepts

**Vấn đề:** "It works on my machine" — code chạy tốt trên laptop nhưng fail khi deploy.

**Nguyên nhân:**
- Hardcoded secrets
- Khác biệt về environment (Python version, OS, dependencies)
- Không có health checks
- Config không linh hoạt

**Giải pháp:** 12-Factor App principles

###  Exercise 1.1: Phát hiện anti-patterns

```bash
cd 01-localhost-vs-production/develop
```

**Nhiệm vụ:** Đọc `app.py` và tìm ít nhất 5 vấn đề.

<details>
<summary> Gợi ý</summary>

Tìm:
- API key hardcode
- Port cố định
- Debug mode
- Không có health check
- Không xử lý shutdown

</details>

###  Exercise 1.2: Chạy basic version

```bash
pip install -r requirements.txt
python app.py
```

Test:
```bash
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

**Quan sát:** Nó chạy! Nhưng có production-ready không?

###  Exercise 1.3: So sánh với advanced version

```bash
cd ../production
cp .env.example .env
pip install -r requirements.txt
python app.py
```

**Nhiệm vụ:** So sánh 2 files `app.py`. Điền vào bảng:

| Feature      | Basic                                                                                                | Advanced                                                                                                            | Tại sao quan trọng?                                                                                                                                                |
| ------------ | ---------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Config       | Hardcode secrets và config ngay trong code (`OPENAI_API_KEY`, `DATABASE_URL`, `DEBUG`, `MAX_TOKENS`) | Dùng environment/config object qua `settings` (`app_name`, `port`, `host`, `debug`, `allowed_origins`, `llm_model`) | Giúp an toàn hơn, không lộ secret khi push code, dễ đổi cấu hình theo môi trường dev/staging/prod mà không sửa mã nguồn.                                           |
| Health check | Không có endpoint health check                                                                       | Có `/health`, thêm cả `/ready` và `/metrics`                                                                        | Cloud platform và load balancer cần endpoint này để biết app còn sống không, đã sẵn sàng nhận traffic chưa, và khi nào cần restart hoặc tạm ngừng route request.   |
| Logging      | Dùng `print()` và còn in cả secret ra log                                                            | Dùng structured JSON logging với `logging`, log event có cấu trúc, tránh log secrets                                | Log có cấu trúc giúp dễ tìm kiếm, phân tích, đưa vào Datadog/Loki/ELK; đồng thời tránh rò rỉ thông tin nhạy cảm trong production.                                  |
| Shutdown     | Đột ngột, không có lifecycle/shutdown handling                                                       | Graceful shutdown với `lifespan`, readiness flag, và xử lý `SIGTERM`                                                | Khi deploy trên cloud/container, app cần đóng kết nối và cho request đang chạy hoàn thành trước khi tắt, tránh mất dữ liệu hoặc lỗi giữa chừng.                    |


###  Checkpoint 1

- [ ] Hiểu tại sao hardcode secrets là nguy hiểm
- [ ] Biết cách dùng environment variables
- [ ] Hiểu vai trò của health check endpoint
- [ ] Biết graceful shutdown là gì

---

## Part 2: Docker Containerization (45 phút)

###  Concepts

**Vấn đề:** "Works on my machine" part 2 — Python version khác, dependencies conflict.

**Giải pháp:** Docker — đóng gói app + dependencies vào container.

**Benefits:**
- Consistent environment
- Dễ deploy
- Isolation
- Reproducible builds

###  Exercise 2.1: Dockerfile cơ bản

```bash
cd ../../02-docker/develop
```

**Nhiệm vụ:** Đọc `Dockerfile` và trả lời:

1. Base image là gì?
Base image là python:3.11. Đây là image Python đầy đủ, được chọn ở dòng FROM python:3.11.
2. Working directory là gì?
Working directory là /app, được đặt bằng WORKDIR /app. Từ sau dòng này, các lệnh như COPY, RUN, CMD sẽ mặc định làm việc trong thư mục đó nếu dùng đường dẫn tương đối.
3. Tại sao COPY requirements.txt trước?
Vì để tận dụng Docker layer cache. Dependencies thường thay đổi ít hơn source code, nếu requirements.txt không đổi, Docker có thể reuse layer pip install, image build lại nhanh hơn nhiều vì không cần cài lại toàn bộ package.
4. CMD vs ENTRYPOINT khác nhau thế nào?
- CMD là lệnh mặc định chạy khi container start. Nó dễ bị override khi truyền lệnh khác lúc docker run. Trong file này, CMD ["python", "app.py"] nghĩa là mặc định container sẽ chạy app bằng Python.
- ENTRYPOINT là điểm vào cố định của container. Nó biến container thành “một executable”, và các đối số thêm vào khi docker run thường được nối vào sau ENTRYPOINT thay vì thay thế hoàn toàn.

###  Exercise 2.2: Build và run

```bash
# Build image
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .

# Run container
docker run -p 8000:8000 my-agent:develop

# Test
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```

**Quan sát:** Image size là bao nhiêu?
```bash
docker images my-agent:develop
REPOSITORY   TAG       IMAGE ID       CREATED          SIZE
my-agent     develop   f3ff076db089   11 minutes ago   1.66GB
```

###  Exercise 2.3: Multi-stage build

```bash
cd ../production
```

**Nhiệm vụ:** Đọc `Dockerfile` và tìm:
- Stage 1 (builder): cài dependencies và build các package cần compile.
- Stage 2 (runtime): tạo image production chỉ chứa code và package cần để chạy app.
- Vì sao image nhỏ hơn: vì image cuối không chứa compiler, build tools và các file tạm của quá trình build.

Build và so sánh:
```bash
docker build -t my-agent:advanced .
docker images | grep my-agent

REPOSITORY                                      TAG                       IMAGE ID       CREATED          SIZE
my-agent                                        advanced                  b8afee794b48   10 minutes ago   236MB
my-agent                                        develop                   f3ff076db089   36 minutes ago   1.66GB
docker.elastic.co/kibana/kibana                 8.17.3                    7dfee7a14cf7   13 months ago    1.92GB
docker.elastic.co/elasticsearch/elasticsearch   8.17.3                    224c75e346bd   13 months ago    2.05GB
bde2020/hadoop-resourcemanager                  2.0.0-hadoop3.2.1-java8   91ca2afe8616   6 years ago      2.05GB
bde2020/hadoop-namenode                         2.0.0-hadoop3.2.1-java8   51ad9293ec52   6 years ago      2.05GB
bde2020/hadoop-datanode                         2.0.0-hadoop3.2.1-java8   ddf6e9ad55af   6 years ago      2.05GB
my-agent                                        advanced                  b8afee794b48   10 minutes ago   236MB
my-agent                                        develop                   f3ff076db089   36 minutes ago   1.66GB
```

###  Exercise 2.4: Docker Compose stack

**Nhiệm vụ:** Đọc `docker-compose.yml` và vẽ architecture diagram. 

```bash
docker compose up
```
Internet / Browser / curl
          |
          v
     +---------+
     |  nginx  |
     | :80 :443|
     +---------+
          |
          | reverse proxy
          v
     +---------+
     |  agent  |
     | FastAPI |
     | :8000   |
     +---------+
       |     |
       |     |
       v     v
   +-------+  +--------+
   | redis |  | qdrant |
   | :6379 |  | :6333  |
   +-------+  +--------+

Network: internal (bridge)
Volumes:
- redis_data   -> Redis persistence
- qdrant_data  -> Qdrant persistence

Services nào được start? Chúng communicate thế nào?

Khi docker compose up, các service được start là: nginx, agent, redis, qdrant.
Chúng giao tiếp như sau:
- Client → nginx qua localhost port 80/443
- nginx → agent bằng reverse proxy trong mạng nội bộ
- agent → redis để cache/session/rate limit
- agent → qdrant để vector search / RAG
Tất cả cùng giao tiếp qua Docker network nội bộ bằng service name

Test:
```bash
# Health check
curl http://localhost/health

# Agent endpoint
curl http://localhost/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'
```

###  Checkpoint 2

- [ ] Hiểu cấu trúc Dockerfile
- [ ] Biết lợi ích của multi-stage builds
- [ ] Hiểu Docker Compose orchestration
- [ ] Biết cách debug container (`docker logs`, `docker exec`)

---

## Part 3: Cloud Deployment (45 phút)

###  Concepts

**Vấn đề:** Laptop không thể chạy 24/7, không có public IP.

**Giải pháp:** Cloud platforms — Railway, Render, GCP Cloud Run.

**So sánh:**

| Platform | Độ khó | Free tier | Best for |
|----------|--------|-----------|----------|
| Railway | ⭐ | $5 credit | Prototypes |
| Render | ⭐⭐ | 750h/month | Side projects |
| Cloud Run | ⭐⭐⭐ | 2M requests | Production |

###  Exercise 3.1: Deploy Railway (15 phút)

```bash
cd ../../03-cloud-deployment/railway
```

**Steps:**

1. Install Railway CLI:
```bash
npm i -g @railway/cli
```

2. Login:
```bash
railway login
```

3. Initialize project:
```bash
railway init
```

4. Set environment variables:
```bash
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
```

5. Deploy:
```bash
railway up
```

6. Get public URL:
```bash
railway domain
```

**Nhiệm vụ:** Test public URL với curl hoặc Postman.

Test:
```bash
# Health check
curl http://student-agent-domain/health

status uptime_seconds platform timestamp                       
------ -------------- -------- ---------                       
ok              743.9 Railway  2026-04-17T15:39:18.458848+00:00

# Agent endpoint
curl http://studen-agent-domain/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": ""}'

  question              answer                                                                                                            platfor
                                                                                                                                        m      
--------              ------                                                                                                            -------
Explain microservices Ä¢y lÃ  cÃ¢u tráº£ lá»Ã¢y sáº½ lÃ  response tá»« OpenAI/Anthropic. Railway
```

###  Exercise 3.2: Deploy Render (15 phút)

```bash
cd ../render
```

**Steps:**

1. Push code lên GitHub (nếu chưa có)
2. Vào [render.com](https://render.com) → Sign up
3. New → Blueprint
4. Connect GitHub repo
5. Render tự động đọc `render.yaml`
6. Set environment variables trong dashboard
7. Deploy!

**Nhiệm vụ:** So sánh `render.yaml` với `railway.toml`. Khác nhau gì?
- render.yaml là file Blueprint/IaC, mô tả cả hạ tầng trên Render như web service, Redis, env vars, region, plan, auto deploy trong một file YAML.
- railway.toml là file cấu hình deploy cho một service trên Railway, chủ yếu khai báo builder, start command, health check, restart policy; biến môi trường thường set thêm qua Dashboard hoặc CLI.

###  Exercise 3.3: (Optional) GCP Cloud Run (15 phút)

```bash
cd ../production-cloud-run
```

**Yêu cầu:** GCP account (có free tier).

**Nhiệm vụ:** Đọc `cloudbuild.yaml` và `service.yaml`. Hiểu CI/CD pipeline.

###  Checkpoint 3

- [ ] Deploy thành công lên ít nhất 1 platform
- [ ] Có public URL hoạt động
- [ ] Hiểu cách set environment variables trên cloud
- [ ] Biết cách xem logs

---

## Part 4: API Security (40 phút)

###  Concepts

**Vấn đề:** Public URL = ai cũng gọi được = hết tiền OpenAI.

**Giải pháp:**
1. **Authentication** — Chỉ user hợp lệ mới gọi được
2. **Rate Limiting** — Giới hạn số request/phút
3. **Cost Guard** — Dừng khi vượt budget

###  Exercise 4.1: API Key authentication

```bash
cd ../../04-api-gateway/develop
```

**Nhiệm vụ:** Đọc `app.py` và tìm:
- API key được check ở đâu?
- Điều gì xảy ra nếu sai key?
- Làm sao rotate key?

Ứng dụng kiểm tra API key trong dependency verify_api_key() thông qua header X-API-Key. Nếu thiếu key sẽ trả 401, nếu sai key sẽ trả 403. Việc rotate key được thực hiện bằng cách đổi biến môi trường AGENT_API_KEY và redeploy/restart ứng dụng.

Test:
```bash
python app.py

#  Không có key
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'

#  Có key
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: secret-key-123" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

###  Exercise 4.2: JWT authentication (Advanced)

```bash
cd ../production
```

**Nhiệm vụ:** 
1. Đọc `auth.py` — hiểu JWT flow
2. Lấy token:
```bash
python app.py

curl http://localhost:8000/token -X POST \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "secret"}'

  >>   -Method POST `
>>   -ContentType "application/json" `
>>   -Body '{"username":"student","password":"demo123"}'
>> $resp

access_token                                                                                                                       
------------                                                                                                                       
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJzdHVkZW50Iiwicm9sZSI6InVzZXIiLCJpYXQiOjE3NzY0NDI1NzksImV4cCI6MTc3NjQ0NjE3OX0.aUu...
```

3. Dùng token để gọi API:
```bash
TOKEN="<token_từ_bước_2>"
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain JWT"}'

  question    answer                                                                                     usage                       
--------    ------                                                                                     -----                       
Explain JWT TÃ´i lÃ  AI agent ÄÆ°á»£c deploy lÃªn cloud. CÃ¢u há»i cá»§a báº¡n ÄÃ£ ÄÆ°á»£c nháºn. @{requests_remaining=9; b...
```

###  Exercise 4.3: Rate limiting

**Nhiệm vụ:** Đọc `rate_limiter.py` và trả lời:
- Algorithm nào được dùng? (Token bucket? Sliding window?)
Algorithm được dùng: Sliding Window Counter. Mỗi user có một bucket deque lưu timestamp request trong cửa sổ 60 giây; request cũ bị loại ra khỏi window trước khi kiểm tra limit.
- Limit là bao nhiêu requests/minute?
User thường: 10 request / 60 giây
Admin: 100 request / 60 giây
- Làm sao bypass limit cho admin?
Trong app.py, hệ thống chọn limiter theo role: nếu role == "admin" thì dùng rate_limiter_admin, ngược lại dùng rate_limiter_user. Vì vậy admin không bypass hoàn toàn, nhưng có ngưỡng cao hơn rất nhiều.
Test:
```bash
# Gọi liên tục 20 lần
for i in {1..20}; do
  curl http://localhost:8000/ask -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"question": "Test '$i'"}'
  echo ""
done


Quan sát response khi hit limit.
> (Gửi liên tục 11 request trong 1 phút)

Request 1-10: HTTP 200 OK
Request 11:   HTTP 429 Too Many Requests
{"detail":"Lỗi 429: Bạn đã hỏi quá 10 lần/phút! Hãy hít thở vài giây nhé."}
```
→ Kết quả: Sau 10 request, server trả 429 đúng như thiết kế chặn spam.

### Exercise 4.4: Cost guard implementation
- Dùng Redis để ghi nợ số tiền của từng `user_id` gán với tháng hiện tại (`YYYY-MM`).
- Sau mỗi câu trả lời, cộng $0.05 USD vào Key Redis (`budget:<user_id>:<YYYY-MM>`).
- Khi tổng nợ của tài khoản này > $10.0 USD, trả về mã lỗi HTTP `402 Payment Required`.
- Key Redis tự hết hạn sau 32 ngày (đầu tháng sau user được "cấp vốn mới").

**Luồng xử lý trong `cost_guard.py`:**
1. Tính `estimated_cost = 0.05` cho mỗi request.
2. Đọc `current_spent` từ Redis key `budget:{user_id}:{YYYY-MM}`.
3. Nếu `current_spent + estimated_cost > monthly_budget_usd` → raise `HTTPException(402)`.
4. Nếu chưa vượt → `r.incrbyfloat(key, 0.05)` để cập nhật số dư.
5. `r.expire(key, 32 * 24 * 3600)` để tự xóa sau 32 ngày.

## Part 5: Scaling & Reliability

### Exercise 5.1: Health Check & Readiness Check

**Health Check (`/health`):**
- Endpoint đơn giản trả về `{"status": "ok", "uptime_seconds": X}`.
- Cloud platform (Railway/Kubernetes) liên tục gọi endpoint này để xác định app còn sống không.
- Nếu không phản hồi → tự động restart container.

**Readiness Check (`/ready`):**
- Kiểm tra kết nối Redis bằng `r.ping()`.
- Nếu Redis chưa sẵn sàng → trả 503 Service Unavailable.
- Load Balancer sẽ không route traffic vào instance chưa ready.

```
> curl https://day11aithucchien-production.up.railway.app/ready

# Nếu Redis đã kết nối:
{"status":"ready"}

# Nếu Redis chưa sẵn sàng:
HTTP 503: {"detail":"Đang khởi động Database, vui lòng đợi (503 Service Unavailable)"}
```

### Exercise 5.2: Graceful Shutdown

Implement trong `main.py` bằng module `signal`:

```python
def shutdown_handler(signum, frame):
    print("Nhận tín hiệu tắt máy (SIGTERM)...")
    sys.exit(0)
signal.signal(signal.SIGTERM, shutdown_handler)
```

**Tại sao quan trọng?**
- Khi Cloud deploy phiên bản mới (rolling update), nó gửi `SIGTERM` tới container cũ.
- Nếu không bắt tín hiệu → request đang xử lý dở sẽ bị cắt ngang → lỗi cho user.
- Graceful shutdown cho phép hoàn thành các request hiện tại trước khi tắt.

### Exercise 5.3: Stateless Design

- **Nguyên tắc:** Tuyệt đối KHÔNG lưu session/state trong RAM của Python process.
- **Thực hiện:** Mọi dữ liệu tạm (rate limit counter, cost tracking) đều lưu vào Redis.
- **Lợi ích:** Khi scale horizontal (chạy nhiều container), tất cả instance chia sẻ chung bộ nhớ Redis → dữ liệu nhất quán.

```
                    ┌─── Container 1 ───┐
User ──→ LB ──────→│   app (stateless)  │──→ Redis (shared state)
                    ├─── Container 2 ───┤        ↑
                    │   app (stateless)  │────────┘
                    └───────────────────┘
```

### Exercise 5.4: Redis là Single Source of Truth

Các key Redis được sử dụng:
- `rate_limit:{user_id}:{minute}` — Đếm số request/phút (TTL: 60s)
- `budget:{user_id}:{YYYY-MM}` — Tổng chi phí tháng (TTL: 32 ngày)

Khi bất kỳ container nào nhận request, nó đều đọc/ghi cùng một Redis → rate limit và cost guard hoạt động chính xác dù có bao nhiêu instance.

### Exercise 5.5: Multi-stage Docker Build

```dockerfile
# Stage 1: Builder — chứa compiler, build tools
FROM python:3.11-slim as builder
RUN pip wheel ... -r requirements.txt

# Stage 2: Production — chỉ copy .whl files, vứt bỏ rác
FROM python:3.11-slim
COPY --from=builder /app/wheels /wheels
RUN pip install /wheels/*
```

**Kết quả:** Image production chỉ ~100-300MB thay vì ~1GB, tiết kiệm bandwidth và thời gian deploy.
