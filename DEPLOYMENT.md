# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Bùi Đức Thành |
| Mã học viên | 2A202602364 |
| Repo | https://github.com/n4hhh/K4-L3B-DAY12-BuiDucThanh-2A202602364-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-agent-production-5931.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-29 |

## Biến Môi Trường Đã Set Trên Cloud

Các biến môi trường được cấu hình trên Railway. Giá trị secret không được lưu trong repository.

| Biến | Trạng thái | Ghi chú |
|------|------------|---------|
| `PORT` | Đã set | Railway tự cấp cổng cho service |
| `AGENT_API_KEY` | Đã set | Secret được lưu trong Railway Variables, không lưu trong repo |
| `REDIS_URL` | Đã set | Reference tới Railway Redis service |
| `RATE_LIMIT_PER_MINUTE` | Đã set | Giá trị 10 |
| `MONTHLY_BUDGET_USD` | Đã set | Giá trị 10.0 |
| `LOG_LEVEL` | Đã set | INFO |

## Kết Quả Chạy Thật

### Liveness Check

Request:

```bash
curl -i https://day12-agent-production-5931.up.railway.app/health
```

Kết quả:

```text
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ok","service":"day12-agent","version":"1.0.0"}
```

### Readiness Check

Request:

```bash
curl -i https://day12-agent-production-5931.up.railway.app/ready
```

Kết quả:

```text
HTTP/1.1 200 OK
Content-Type: application/json

{"status":"ready","redis":true}
```

Kết quả này xác nhận web service đang hoạt động và đã kết nối thành công tới Redis trên Railway.

### Authentication Check

Request không có API key:

```bash
curl -i -X POST https://day12-agent-production-5931.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
```

Kết quả mong đợi:

```text
HTTP/1.1 401 Unauthorized
```

Endpoint `/ask` yêu cầu API key hợp lệ thông qua header `X-API-Key`.

### Authenticated Request

Request hợp lệ sử dụng API key được lưu trong biến môi trường cục bộ, không ghi trực tiếp secret vào tài liệu:

```bash
curl -i -X POST https://day12-agent-production-5931.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'
```

Khi API key hợp lệ, endpoint trả HTTP 200 cùng câu trả lời của agent.

## Kiến Trúc Deployment

Service được triển khai trên Railway bằng Dockerfile của repository.

Các thành phần chính:

- `day12-agent`: FastAPI web service.
- `Redis`: Railway Redis service dùng để lưu conversation history, rate-limit state và cost tracking.
- Public networking: Railway HTTPS domain.
- Health endpoint: `/health`.
- Readiness endpoint: `/ready`.

Web service sử dụng `REDIS_URL` để kết nối tới Redis service nội bộ trên Railway.

## Bảo Mật

`AGENT_API_KEY` không được hardcode trong source code, Dockerfile, docker-compose hoặc tài liệu.

Secret được lưu trong Railway Variables và được inject vào container khi service chạy.

File `.env` local không được commit vào Git repository.

## Public Deployment

Public service:

```text
https://day12-agent-production-5931.up.railway.app
```

Deployment đang sử dụng Railway cloud và không sử dụng phương án local fallback.