# LAB 04 - DOCKER PACKAGING & NEWMAN VERIFICATION
## TÀI LIỆU MINH CHỨNG NỘP BÀI (SUBMISSION EVIDENCE)

Tài liệu này tổng hợp toàn bộ 6 thành phần cần nộp cho Lab 04 theo quy ước chuẩn của lớp học để bạn dễ dàng sao chép và nộp bài.

---

### 1. Dockerfile
Nội dung file [Dockerfile](file:///d:/DVCNNTANG/lab-04-ngocanh616/Dockerfile) tối ưu hóa nhiều tầng (Multi-stage build), chạy non-root user và tích hợp Healthcheck:

```dockerfile
# syntax=docker/dockerfile:1.7

FROM python:3.11-slim AS builder

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

WORKDIR /build

RUN python -m venv /opt/venv

COPY requirements.txt .

RUN /opt/venv/bin/pip install --no-cache-dir --upgrade pip \
    && /opt/venv/bin/pip install --no-cache-dir -r requirements.txt


FROM python:3.11-slim AS runtime

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH="/opt/venv/bin:$PATH"
ENV APP_HOST=0.0.0.0
ENV APP_PORT=8000
ENV AUTH_TOKEN=local-dev-token

WORKDIR /app

RUN addgroup --system appgroup \
    && adduser --system --ingroup appgroup --home /app appuser

COPY --from=builder /opt/venv /opt/venv
COPY src/ ./src/

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD python -c "import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health', timeout=3).read()" || exit 1

CMD ["sh", "-c", "uvicorn iot_app.main:app --app-dir src --host ${APP_HOST} --port ${APP_PORT}"]
```

---

### 2. .dockerignore
Nội dung file [.dockerignore](file:///d:/DVCNNTANG/lab-04-ngocanh616/.dockerignore) loại bỏ các thư mục rác giúp dung lượng build context nhẹ nhất:

```text
.git
.github
.venv
__pycache__
*.pyc
*.pyo
*.pyd
.Python
.env
.env.*
!.env.example
node_modules
reports
*.log
.DS_Store
.cache
.pytest_cache
dist
build
.coverage
htmlcov
```

---

### 3. .env.example
Nội dung file [.env.example](file:///d:/DVCNNTANG/lab-04-ngocanh616/.env.example) dùng để khai báo biến cấu hình runtime:

```text
APP_HOST=0.0.0.0
APP_PORT=8000
AUTH_TOKEN=local-dev-token
SERVICE_NAME=iot-ingestion
SERVICE_VERSION=0.4.0
ENV=local
```

---

### 4. RUN_LOCAL.md
Nội dung file [RUN_LOCAL.md](file:///d:/DVCNNTANG/lab-04-ngocanh616/RUN_LOCAL.md) đã được chuẩn hóa liên kết Repo của bạn:

```markdown
# RUN_LOCAL.md – Hướng dẫn chạy Lab 04

Tài liệu này giúp người khác clone repo sạch và chạy lại service trong Docker.

---

## 1. Clone repo

```bash
git clone https://github.com/Connectivity-services-ad-PT/lab-04-ngocanh616.git
cd lab-04-ngocanh616
```

---

## 2. Cài dependencies cho Newman/Prism/Spectral

```bash
npm install
```

---

## 3. Build Docker image

```bash
docker build -t fit4110/iot-ingestion:lab04 .
```

---

## 4. Run container

```bash
docker run --rm \
  --name fit4110-iot-lab04 \
  -p 8000:8000 \
  --env-file .env.example \
  fit4110/iot-ingestion:lab04
```

Mở terminal khác, kiểm tra:

```bash
curl http://localhost:8000/health
```

*Lưu ý cho người dùng Windows PowerShell:* Thay `curl` bằng `curl.exe` hoặc `Invoke-RestMethod http://localhost:8000/health` để tránh cảnh báo bảo mật script.

Kết quả mong đợi:

```json
{
  "status": "ok",
  "service": "iot-ingestion",
  "version": "0.4.0"
}
```

---

## 5. Chạy Newman test trên container

```bash
npm run test:local
```

Report sinh tại:

```text
reports/newman-lab04-local.xml
reports/newman-lab04-local.html
```

---

## 6. Dừng container

Nếu không dùng `--rm` hoặc container còn chạy:

```bash
docker stop fit4110-iot-lab04
```

---

## 7. Lệnh nhanh

```bash
make build
make run
make test-docker
make stop
```
```

---

### 5. Log thực thi cục bộ (build-run-test)

#### a) Log Docker Build
```bash
$ docker build -t fit4110/iot-ingestion:lab04 .
#0 building with "desktop-linux" instance using docker driver
#1 [internal] load build definition from Dockerfile
#1 DONE 0.1s
#2 [internal] load .dockerignore
#2 DONE 0.1s
#3 [internal] load metadata for docker.io/library/python:3.11-slim
#3 DONE 1.7s
#4 [builder 1/5] FROM docker.io/library/python:3.11-slim
#4 DONE 0.1s
...
#17 naming to docker.io/fit4110/iot-ingestion:lab04 done
#17 DONE 0.5s
```

#### b) Log Docker Run (Khởi động Service non-root)
```bash
$ docker run -d --rm --name fit4110-iot-lab04 -p 8000:8000 --env-file .env.example fit4110/iot-ingestion:lab04
28734961a659cdde7fea39bc66b36691b9c37343e5358d8ead8005845eb68868

$ docker logs fit4110-iot-lab04
INFO:     Started server process [7]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

#### c) Log Curl Health Check
```powershell
$ Invoke-RestMethod -Uri http://localhost:8000/health

status service       version
------ -------       -------
ok     iot-ingestion 0.4.0  
```

#### d) Log Newman Test (Đầy đủ 4 nhóm test với 19 Assertions vượt qua 100%)
```bash
$ npm run test:local

> fit4110-lab04-docker-packaging@0.1.0 test:local
> newman run postman/collections/FIT4110_lab04_iot_docker.postman_collection.json -e postman/environments/FIT4110_lab04_local.postman_environment.json -r cli,junit,htmlextra --reporter-junit-export reports/newman-lab04-local.xml --reporter-htmlextra-export reports/newman-lab04-local.html

newman

FIT4110 Lab04 IoT Docker Verification

□ 01_Functional
└ GET health returns 200
  GET http://localhost:8000/health [200 OK, 184B, 118ms]
  √  Status code is 200
  √  Response has status ok
  √  Response has service name and version

└ POST valid temperature reading returns 201
  POST http://localhost:8000/readings [201 Created, 271B, 229ms]
  √  Status code is 201
  √  Response follows created-reading schema
  √  Response device_id matches request

└ GET latest readings returns items array
  GET http://localhost:8000/readings/latest?device_id=ESP32-LAB-A01&limit=5 [200 OK, 332B, 15ms]
  √  Status code is 200
  √  Response has items array

└ GET reading by saved reading_id returns 200
  GET http://localhost:8000/readings/R-20260526-0001 [200 OK, 320B, 11ms]
  √  Status code is 200
  √  Response reading_id matches saved variable

□ 02_Auth
└ POST reading without token returns 401
  POST http://localhost:8000/readings [401 Unauthorized, 302B, 17ms]
  √  Missing token returns 401

└ POST reading with wrong token returns 401
  POST http://localhost:8000/readings [401 Unauthorized, 294B, 8ms]
  √  Wrong token returns 401

□ 03_Negative
└ POST reading missing device_id returns validation error
  POST http://localhost:8000/readings [422 Unprocessable Entity, 320B, 13ms]
  √  Missing required field returns 422

└ POST reading with value as string returns validation error
  POST http://localhost:8000/readings [422 Unprocessable Entity, 368B, 9ms]
  √  Wrong data type returns 422

□ 04_Boundary_Reliability
└ POST boundary temperature 80 is accepted with warning
  POST http://localhost:8000/readings [201 Created, 300B, 7ms]
  √  Boundary value 80 returns 201
  √  High temperature response includes warning header

└ POST boundary temperature 81 is rejected
  POST http://localhost:8000/readings [422 Unprocessable Entity, 342B, 7ms]
  √  Boundary value 81 returns 422

└ GET health responds under 1000ms on local/container
  GET http://localhost:8000/health [200 OK, 184B, 4ms]
  √  Response time is below 1000ms
  √  Health endpoint is reachable

┌─────────────────────────┬───────────────────┬──────────────────┐
│                         │          executed │           failed │
├─────────────────────────┼───────────────────┼──────────────────┤
│              iterations │                 1 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│                requests │                11 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│            test-scripts │                11 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│      prerequest-scripts │                 0 │                0 │
├─────────────────────────┼───────────────────┼──────────────────┤
│              assertions │                19 │                0 │
├─────────────────────────┴───────────────────┴──────────────────┤
│ total run duration: 1391ms                                     │
├────────────────────────────────────────────────────────────────┤
│ total data received: 1.68kB (approx)                           │
├────────────────────────────────────────────────────────────────┤
│ average response time: 39ms [min: 4ms, max: 229ms, s.d.: 67ms] │
└────────────────────────────────────────────────────────────────┘
```

---

### 6. Thông tin Image đã Gắn Nhãn (Tag) & Đẩy Lên (Push)
Quy ước gắn nhãn cho team của bạn đã được thực hiện cục bộ thành công:

```bash
# Tag image
docker tag fit4110/iot-ingestion:lab04 ghcr.io/connectivity-services-ad-pt/team-iot:v0.1.0-team-iot
docker tag fit4110/iot-ingestion:lab04 ghcr.io/ngocanh616/team-iot:v0.1.0-team-iot

# Xem danh sách image
$ docker images
IMAGE                                                          TAG                 IMAGE ID       CREATED        SIZE
ghcr.io/connectivity-services-ad-pt/team-iot                   v0.1.0-team-iot     bf9e738a29fc   1 hour ago     268MB
ghcr.io/ngocanh616/team-iot                                    v0.1.0-team-iot     bf9e738a29fc   1 hour ago     268MB
```

---
*Minh chứng đã được tổng hợp trọn vẹn và tối ưu nhất để nộp bài!*
