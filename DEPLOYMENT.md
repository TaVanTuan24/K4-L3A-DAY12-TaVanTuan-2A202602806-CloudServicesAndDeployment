# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị API key vào đây.**
> Repo này công khai — dán khóa vào là mất khóa.

## Thông Tin Học Viên

| Mục | Nội dung |
|-----|----------|
| Họ và tên | Tạ Văn Tuấn |
| Mã học viên | 2A202602806 |
| Repo | https://github.com/TaVanTuan24/K4-L3A-DAY12-TaVanTuan-2A202602806-CloudServicesAndDeployment |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://production-ai-agent-day12-production.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-09-28 |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `AGENT_API_KEY` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | variable reference `${{Redis.REDIS_URL}}` (Redis nội bộ Railway) |
| `RATE_LIMIT_PER_MINUTE` | ✅ | 10 |
| `MONTHLY_BUDGET_USD` | ✅ | 10.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Sau khi deploy, thay `$BASE_URL` bằng Public URL thật ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i $BASE_URL/health

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i $BASE_URL/ready

# 3. Không có API key — mong đợi 401
curl -i -X POST $BASE_URL/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'

# 4. Có API key — mong đợi 200 kèm câu trả lời
curl -i -X POST $BASE_URL/ask \
  -H "Content-Type: application/json" \
  -H "X-API-Key: $AGENT_API_KEY" \
  -H "X-User-Id: sv-test" \
  -d '{"question":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST $BASE_URL/ask \
    -H "Content-Type: application/json" \
    -H "X-API-Key: $AGENT_API_KEY" \
    -H "X-User-Id: sv-test" \
    -d '{"question":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Output thật thu được khi gọi vào `https://production-ai-agent-day12-production.up.railway.app`:

```text
# 1. Liveness — /health
HTTP 200
{"status":"ok","service":"day12-agent","version":"1.0.0"}

# 2. Readiness — /ready (đã nối được Redis nội bộ Railway)
HTTP 200
{"status":"ready","redis":true}

# 3. Không có API key — /ask
HTTP 401
{"detail":"invalid or missing API key"}

# 4. Có API key — /ask (X-User-Id: sv-test)
HTTP 200
{"answer":"Ngắn gọn: Deploy la gi phụ thuộc vào ba yếu tố — cấu hình qua biến môi trường, health check để orchestrator biết trạng thái, và giới hạn tài nguyên.","user_id":"sv-test","history_length":0,"cost_usd":2.265e-05,"tokens":{"in":3,"out":37}}

# 5. Rate limit — 15 request liên tiếp (X-User-Id: rate-limit-test)
200 200 200 200 200 200 200 200 200 200 429 429 429 429 429

# 6. History / stateless — 3 request liên tiếp cùng X-User-Id: history-test
history_length = 0 → 2 → 4
```

## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên Railway (chụp sau khi deploy thật)
- `screenshots/health.png` — kết quả gọi `/health` từ trình duyệt hoặc curl (chụp sau khi deploy thật)

> Đã deploy thật (Railway). Màn hình chưa có (agent không mở được trình duyệt để chụp
> ảnh) — cần tự chụp dashboard Railway và ảnh gọi `/health` rồi đặt vào `screenshots/`.

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

```
Không áp dụng phương án dự phòng — sử dụng deploy thật lên Railway (xem phần "Service" ở trên).
```
