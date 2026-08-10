# Thông Tin Deploy — Checkpoint 5

> Điền file này sau khi deploy xong. `pytest tests/test_cp5.py` đọc file này
> để tìm địa chỉ service của bạn và gọi thử.
>
> **Chỉ ghi TÊN biến môi trường, tuyệt đối không dán giá trị token vào đây.**
> Repo này công khai — dán token vào là mất token.

## Thông Tin Học Viên

| Họ và tên | Nhat Anh |
| Mã học viên | K4-Day12 |
| Repo | LOCAL-FALLBACK |

## Service

| Mục | Nội dung |
|-----|----------|
| Public URL | https://day12-chat-production-f2f6.up.railway.app |
| Platform | Railway |
| Ngày deploy | Hôm nay |

## Biến Môi Trường Đã Set Trên Cloud

Ghi tên biến và **nguồn giá trị**, không ghi giá trị:

| Biến | Đã set | Ghi chú |
|------|--------|---------|
| `PORT` | ✅ | platform tự gán |
| `API_TOKEN` | ✅ | đặt trong dashboard, không nằm trong repo |
| `REDIS_URL` | ✅ | Railway Redis Database |
| `BUCKET_CAPACITY` | ✅ | 10 |
| `REFILL_PER_MINUTE` | ✅ | 10 |
| `DAILY_BUDGET_USD` | ✅ | 1.0 |
| `LOG_LEVEL` | ✅ | INFO |

## Lệnh Kiểm Tra

Thay `<URL>` bằng Public URL ở trên:

```bash
# 1. Liveness — mong đợi 200 {"status":"ok"}
curl -i <URL>/healthz

# 2. Readiness — mong đợi 200 {"status":"ready"} (đã nối được Redis)
curl -i <URL>/readyz

# 3. Không có token — mong đợi 401 kèm header WWW-Authenticate
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# 4. Có token — mong đợi 200 kèm câu trả lời
curl -i -X POST <URL>/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'

# 5. Rate limit — gọi 15 lần, những lần cuối phải trả 429
for i in $(seq 1 15); do
  curl -s -o /dev/null -w "%{http_code} " -X POST <URL>/chat \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer $API_TOKEN" \
    -H "X-Client-Id: sv-test" \
    -d '{"message":"test"}'
done; echo
```

## Kết Quả Chạy Thật

Dán output của các lệnh trên vào đây:

```text
Output of 5 test commands:
---
# 1. Liveness
HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 16:13:52 GMT
server: railway-hikari

{"status":"ok","service":"day12-chat-service","version":"1.0.0"}


# 2. Readiness
HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 16:13:52 GMT
server: railway-hikari

{"status":"ready","redis":true}


# 3. Không có token
HTTP/2 401 
content-type: application/json
date: Mon, 10 Aug 2026 16:13:53 GMT
server: railway-hikari
www-authenticate: Bearer

{"detail":"invalid or missing bearer token"}


# 4. Có token
HTTP/2 200 
content-type: application/json
date: Mon, 10 Aug 2026 16:13:53 GMT
server: railway-hikari

{"reply":"Câu hỏi hay. Deploy là gì thường được giải quyết bằng cách chuẩn hóa môi trường chạy: cùng một image chạy giống nhau ở laptop và trên cloud.","client_id":"sv-test","turns_before":0,"usd_cost":2.145e-05,"usage":{"prompt":3,"completion":35}}

# 5. Rate limit
200 200 200 200 200 200 200 200 200 429 429 429 429 429 429
```
## Ảnh Chụp Màn Hình

Đặt ảnh trong thư mục `screenshots/`:

- `screenshots/dashboard.png` — trang quản lý service trên platform
- `screenshots/healthz.png` — kết quả gọi `/healthz` từ trình duyệt hoặc curl

---

## Nếu Dùng Phương Án Dự Phòng

Không đăng ký được tài khoản cloud? Vẫn nộp được bài, nhưng CP5 tối đa 60% điểm:

1. Đặt `LOCAL_FALLBACK=true` trong `.env`
2. Chạy `docker compose up -d` rồi kiểm tra `docker compose ps`
3. Chụp màn hình vào `screenshots/`
4. Chạy `pytest tests/test_cp5.py -v` — bộ test sẽ tự chuyển sang kiểm tra
   `http://localhost:8000`
5. Ghi rõ lý do không deploy được vào phần dưới đây:

Thành công trên Railway, lỗi ban đầu đã được fix bằng cách bỏ startCommand trong railway.toml và dùng python -m app.main trong Dockerfile.
