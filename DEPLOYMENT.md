# Thông Tin Deploy — Checkpoint 5

## Thông Tin Học Viên

| Mục | Nội dung |
|---|---|
| Họ và tên | Lê Đình Việt |
| Mã học viên | 2A202601528 |
| Repo | https://github.com/CodeWithDinhViet/K4-Day12-2A202601528-LeDinhViet |

## Service

| Mục | Nội dung |
|---|---|
| Public URL | https://day12-chat-production-3477.up.railway.app |
| Platform | Railway |
| Ngày deploy | 2026-08-10 |
| Project | K4-Day12-2A202601528-LeDinhViet |
| Service | day12-chat |

Railway build `Dockerfile` trên cloud. Máy local không cần chạy Docker.

## Biến Môi Trường Đã Set Trên Cloud

Chỉ liệt kê tên biến và nguồn, không ghi giá trị bí mật:

| Biến | Đã set | Nguồn |
|---|---|---|
| `PORT` | ✅ | Railway tự gán |
| `API_TOKEN` | ✅ | Railway Variables; giá trị bí mật không nằm trong repo |
| `REDIS_URL` | ✅ | Tham chiếu `Redis.REDIS_URL` từ Redis add-on của Railway |
| `BUCKET_CAPACITY` | ✅ | Railway Variables |
| `REFILL_PER_MINUTE` | ✅ | Railway Variables |
| `DAILY_BUDGET_USD` | ✅ | Railway Variables |
| `LOG_LEVEL` | ✅ | Railway Variables |

## Kết Quả Chạy Thật

Kiểm tra ngày 2026-08-10:

```text
GET  /healthz                  -> 200, status=ok
GET  /readyz                   -> 200, status=ready, redis=true
POST /chat (không token)       -> 401
POST /chat (Bearer token đúng) -> 200, reply có nội dung, client_id=cp5-verify
```

Deployment Railway thành công với health check `/healthz`. Domain HTTPS được định
tuyến tới cổng runtime 8080 do Railway cấp qua biến `PORT`.

## Lệnh Kiểm Tra

```bash
URL=https://day12-chat-production-3477.up.railway.app

curl -i "$URL/healthz"
curl -i "$URL/readyz"
curl -i -X POST "$URL/chat" \
  -H "Content-Type: application/json" \
  -d '{"message":"Hello"}'

# API_TOKEN chỉ lấy từ môi trường bí mật, không lưu trong tài liệu/repo.
curl -i -X POST "$URL/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "X-Client-Id: sv-test" \
  -d '{"message":"Deploy là gì?"}'
```

## Ghi Chú

- Redis chạy bằng add-on riêng trên Railway, không dùng `fake://` khi deploy.
- `API_TOKEN` được tạo ngẫu nhiên và chỉ lưu trong Railway Variables.
- Không sử dụng phương án `LOCAL_FALLBACK`; đây là deployment cloud đầy đủ.
