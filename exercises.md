# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: điền nội dung ngay dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Đình Việt  Mã học viên: 2A202601528

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ cụ thể là khi deploy service công khai nhưng quên cấu hình `API_TOKEN` trên
> Railway. Nếu token mặc định là `"changeme"`, service vẫn báo healthy và bất kỳ ai
> đoán được giá trị này đều có thể gọi `/chat`, làm phát sinh chi phí. Khi
> `api_token` là trường bắt buộc, process dừng ngay lúc khởi động, deployment không
> qua health check và tôi phát hiện lỗi cấu hình trước khi service nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log thật từ deployment Railway của tôi là:
> `{"event":"chat_completed","usd_cost":0.00002265,"ts":"2026-08-10T09:17:50.226008+00:00","client_id":"cp5-verify","prompt_tokens":3,"completion_tokens":37}`.
> Từ log có cấu trúc này, tôi có thể (1) lọc/nhóm theo `event` hoặc `client_id` để
> điều tra một client cụ thể, và (2) cộng `usd_cost`, `prompt_tokens` hoặc
> `completion_tokens` để dựng dashboard và cảnh báo chi phí. Một chuỗi `print`
> thông thường không cung cấp các trường ổn định để máy tự động truy vấn và tổng hợp.

---

### Câu 3 — Kích thước image (CP2)

Build hai phiên bản và so sánh kích thước image:

```bash
docker build -f Dockerfile.single -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản               | Dung lượng |
| ----------------- | ---------: |
| 1 stage (bản đầu) |    ~238 MB |
| Multi-stage       |    ~171 MB |

Chênh lệch khoảng **67 MB**, tương đương image multi-stage nhỏ hơn khoảng **28%**.

Phần dung lượng giảm chủ yếu đến từ những thành phần chỉ cần trong quá trình build như compiler, build tools, header của thư viện và các file/cache trung gian khi cài package Python.

Ở bản 1-stage, quá trình build dependency và chạy ứng dụng diễn ra trong cùng một image nên các công cụ phục vụ build có thể vẫn tồn tại trong image cuối.

Ở bản multi-stage, stage đầu dùng để cài và build dependency. Sau đó runtime stage chỉ lấy những thành phần cần thiết như thư viện đã cài trong `/install`, source `app`, `utils` và các file cấu hình cần để chạy service.

Vì vậy production image không phải mang theo toàn bộ công cụ và artifact của giai đoạn build. Kết quả là image nhỏ hơn, thời gian pull/deploy có thể nhanh hơn và attack surface cũng được giảm bớt.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, sửa `app/main.py` không làm thay đổi `requirements.txt`,
> nên các layer base image, `WORKDIR`, `COPY requirements.txt` và `RUN pip install`
> được dùng lại từ cache. Layer `COPY app ./app` và các layer phía sau nó phải tạo
> lại. Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một thay đổi trong source
> cũng làm mất cache của layer copy, kéo theo việc cài lại toàn bộ dependency dù
> `requirements.txt` không đổi, khiến build chậm và tốn băng thông hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code Python có lỗ hổng thực thi lệnh, kẻ tấn công có thể chạy lệnh với UID của
> process trong container. Khi process là root, họ có quyền sửa file hệ thống trong
> container và có nhiều khả năng lợi dụng thêm lỗi runtime, mount hoặc cấu hình sai
> để tác động tới host. `USER appuser` cắt chuỗi ngay sau bước thực thi lệnh: payload
> chỉ có quyền của UID 10001, không được tùy ý sửa các tài nguyên thuộc root. Đây
> không phải ranh giới tuyệt đối, nhưng làm giảm mạnh quyền và phạm vi thiệt hại.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> `WWW-Authenticate: Bearer` cho client biết cơ chế xác thực mà tài nguyên yêu cầu,
> đúng quy ước HTTP/RFC 6750 và giúp thư viện client biết cần gửi Bearer token.
> Service trả cùng thông báo `invalid or missing bearer token` cho thiếu header, sai
> scheme và sai token để không tiết lộ cho kẻ tấn công họ đã đoán đúng phần nào của
> thông tin xác thực. Client hợp lệ vẫn có status 401 và header chuẩn để sửa request,
> trong khi endpoint không trở thành công cụ dò token hoặc cấu hình xác thực.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Do xô bị giới hạn bởi `capacity`, sau 10 phút nó vẫn chỉ có tối đa 10 token, nên
> client gửi được 10 request liên tiếp; request thứ 11 nhận 429. Nếu bỏ
> `min(capacity, ...)`, 10 phút tạo thêm 100 token. Tùy số token còn lại trước lúc
> im lặng, client có thể tích khoảng 100 đến gần 110 token và burst tương ứng thay
> vì chỉ 10. Lỗi này biến thời gian im lặng thành hạn mức tích lũy không giới hạn và
> làm mất mục đích chặn burst của `capacity`.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Với hạn mức 30 USD/tháng, một sự cố bắt đầu lúc 2 giờ sáng có thể tiêu hết gần 30
> USD trước khi bị chặn; client chỉ tự dùng lại được khi bước sang kỳ/tháng mới hoặc
> khi người vận hành can thiệp. Với hạn mức 1 USD/ngày, thiệt hại của ngày đó bị chặn
> quanh 1 USD và khóa chi tiêu dùng key theo ngày UTC. Sang ngày UTC tiếp theo,
> service tự dùng key mới với ngân sách mới nên phục hồi mà không cần reset thủ công.
> Hạn mức ngày vì thế thu nhỏ blast radius của một sự cố kéo dài.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp readiness vào liveness, khi Redis mất kết nối thì cả ba container đồng
> thời báo probe thất bại. Orchestrator hiểu nhầm process đã chết, loại chúng khỏi
> traffic rồi restart cả ba, dù code ứng dụng vẫn chạy bình thường. Các instance mới
> cùng khởi động và tiếp tục không nối được Redis, tạo vòng restart và tải dồn lên
> dependency đang lỗi. Khi tách probe, `/healthz` vẫn 200 nên process không bị giết;
> `/readyz` trả 503 để tạm ngừng nhận traffic. Redis hồi phục thì `/readyz` tự trở lại
> 200 và ba instance được đưa vào phục vụ mà không cần restart hàng loạt.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lần deploy đầu trên Railway build thành công nhưng container lặp lại lỗi
> `Invalid value for '--port': '$PORT' is not a valid integer`. Tôi xem deployment
> log và thấy `startCommand` trong `railway.toml` được chạy trực tiếp nên `$PORT`
> không được shell mở rộng. Tôi xóa `startCommand` override để Railway dùng `CMD`
> của Dockerfile (`sh -c ... --port ${PORT:-8000}`). Bản deploy sau chạy Uvicorn ở
> cổng 8080 do Railway cấp, health check `/healthz` pass. Sau đó tôi cập nhật domain
> route từ cổng 8000 sang 8080; `/healthz` và `/readyz` đều trả 200.
