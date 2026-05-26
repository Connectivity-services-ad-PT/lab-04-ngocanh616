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
