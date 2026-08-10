# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Đỗ Thanh Tùng |
| Mã học viên | 2A202601205 |
| Repo | https://github.com/thanhtungdo2003/DAY12-2A202601205-DoThanhTung |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-2a202601205-dothanhtung.onrender.com |
| Platform | Render |
| Ngày deploy | 10/08/2026  |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Redis add-on của Render (render.yaml service `day12-redis`) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```
https://day12-2a202601205-dothanhtung.onrender.com/health
{
  "status": "ok",
  "service": "day12-agent",
  "version": "1.0.0"
}

https://day12-2a202601205-dothanhtung.onrender.com/ready
{
  "status": "ready",
  "redis": true
}

https://day12-2a202601205-dothanhtung.onrender.com/ask
{
  "answer": "Theo mình hiểu, string liên quan tới cách hệ thống được đóng gói và vận hành. Điểm mấu chốt là tách cấu hình ra khỏi code và giữ service ở trạng thái stateless. (Mình đang nhớ 2 lượt trao đổi trước đó.)",
  "user_id": "anonymous",
  "history_length": 2,
  "cost_usd": 0.00003645,
  "tokens": {
    "in": 43,
    "out": 50
  }
}
```

## Ảnh Chụp Màn Hình

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl

---
