# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> [Câu trả lời của bạn]` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Mai Nhat Anh  Mã học viên: K4-Day12

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Nếu để mặc định `"changeme"`, ứng dụng vẫn khởi động thành công trên môi trường thật dù ta quên cấu hình biến `API_TOKEN`. Khi đó, hacker có thể đoán được token mặc định này để gọi API liên tục miễn phí, làm cạn kiệt ngân sách AI của chúng ta mà không ai hay biết. Tính năng "Fail Fast" đảm bảo ứng dụng chết ngay lập tức lúc boot để báo hiệu cấu hình sai, không tạo cơ hội cho mã lỗi chạy ngầm.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Dòng log JSON: `{"timestamp": "2026-08-10T16:07:13Z", "level": "INFO", "event": "chat_completed", "client_id": "anonymous", "prompt_tokens": 181, "completion_tokens": 46, "usd_cost": 5.475e-05}`

Hai việc làm được với log JSON:
1. Dễ dàng dùng các công cụ quản lý log (như Datadog, Kibana) để **query và lọc** dựa trên thuộc tính, ví dụ: tìm tất cả request có `usd_cost > 0.0001`.
2. Có thể **thống kê và vẽ biểu đồ** tổng số `completion_tokens` hoặc chi phí tổng theo giờ mà không cần phải viết regex phức tạp để bóc tách chuỗi như lệnh `print`.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t chat:single .
docker build -t chat:multi .
docker images | grep chat
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | ~1000 MB |
| Multi-stage | ~150 MB |

Giải thích: Phần dung lượng chênh lệch chính là compiler (GCC), các toolchain build C/C++, cache của `apt-get` và thư mục cache của `pip` sinh ra trong lúc cài các thư viện Python. Nhờ Multi-stage, stage `runtime` chỉ copy duy nhất thư mục `/install` (chứa các file mã máy đã build) sang base image siêu nhẹ `slim`, bỏ lại toàn bộ "rác" của quá trình build.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Khi sửa 1 ký tự trong `app/main.py`: Các layer trước đó (như `COPY requirements.txt .` và `RUN pip install`) sẽ được dùng lại từ cache do file `requirements.txt` không đổi. Chỉ có layer `COPY app/ app/` và các layer phía sau bị mất cache và phải chạy lại.

Nếu đặt `COPY . .` lên trước `RUN pip install`: Lệnh `COPY . .` sẽ chép cả file `app/main.py`. Khi file này thay đổi, checksum của layer COPY thay đổi làm toàn bộ cache Docker từ điểm đó trở đi bị vô hiệu hoá. Kết quả là lệnh `RUN pip install` ngay phía sau sẽ bị ép phải chạy lại, tải và cài lại toàn bộ thư viện từ đầu rất tốn thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: Ứng dụng Python có lổ hổng (vd: RCE), hacker gửi payload đặc biệt để thực thi shell (`/bin/sh`) trong container. Vì container đang chạy bằng quyền `root`, shell đó sẽ thuộc sở hữu của `root`. Từ đây, hacker có thể cài tool, sửa đổi file hệ thống, và nếu có kẽ hở container (container escape), hacker sẽ thâm nhập ra máy host với quyền `root` của máy host.

Lệnh `USER appuser` cắt đứt chuỗi này ngay từ đầu: Dù hacker chiếm được quyền điều khiển shell trong container, họ cũng chỉ có đặc quyền hạn chế của `appuser`. Họ không thể cài thêm phần mềm bằng `apt`, không thể đọc/ghi file hệ thống, chặn đứng nguy cơ leo thang đặc quyền.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

401 kèm header `WWW-Authenticate: Bearer` là quy định bắt buộc của giao thức HTTP nhằm báo cho HTTP Client (như trình duyệt hoặc Postman) biết rằng server yêu cầu xác thực bằng cơ chế Bearer Token chứ không phải Basic Auth hay loại khác.

Việc trả cùng 1 thông báo lỗi nhằm mục đích **bảo mật (Security through obscurity)**: Nếu ta báo quá chi tiết (ví dụ "Sai chữ ký token" hay "Token hết hạn"), kẻ tấn công có thể lợi dụng những thông tin phản hồi khác nhau này để dò tìm ra cách thuật toán mã hoá token hoạt động hoặc biết được một token cụ thể có tồn tại trong hệ thống hay không. Trả lời chung chung giúp chặn đứng phương thức tấn công dò dẫm này.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

Với `capacity=10`, client im lặng 10 phút cũng chỉ gửi được tối đa **10 request** trước khi bị lỗi 429, bởi vì lượng token tích luỹ không bao giờ vượt qua sức chứa tối đa của "xô" (bucket).

Nếu bỏ đoạn `min(capacity, ...)`: Token sẽ được cộng vô hạn mà không bị trần. Im lặng 10 phút (với refill 10/phút) sẽ tích luỹ được 100 token. Khi đó client có thể bắn xả láng (Burst) **100 request** liên tiếp. Việc này phá vỡ tính năng giới hạn lưu lượng Burst của thuật toán Token Bucket, khiến server có nguy cơ bị sập do lượng lớn request ập tới cùng lúc.

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

Giả sử có sự cố gọi liên tục từ 2h sáng:
- Hạn mức $30/tháng: Thiệt hại tối đa trong đêm đó là **$30**. Client sẽ đốt sạch quỹ của cả tháng chỉ trong vài giờ và bị ngắt kết nối. Service chỉ tự hồi phục lại vào **tháng sau** (tức là sập toàn bộ 29 ngày còn lại).
- Hạn mức $1/ngày: Thiệt hại tối đa trong đêm đó chỉ là **$1**. Service sẽ ngừng phục vụ sớm để bảo vệ túi tiền, và sẽ **tự động hồi phục lại vào 0:00 ngày hôm sau** khi budget được reset. Cách này giúp hạn chế sát thương về tiền và giữ cho hệ thống không bị "chết lâm sàng" cả một chu kì dài.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp chung kiểm tra Redis vào `/healthz` (liveness probe), thứ tự sự kiện sẽ là:
1. Redis mất kết nối trong 30 giây.
2. Endpoint `/healthz` của cả 3 container đều báo lỗi (trả 503).
3. Orchestrator (Docker/Kubernetes) sẽ đọc thấy `/healthz` failed liên tục, lầm tưởng rằng tiến trình Python đã treo nên gửi lệnh SIGTERM để **kill và restart lại đồng loạt cả 3 container**.
4. Toàn bộ request đang được xử lý của người dùng bị ngắt ngang.
5. Sau 30s, Redis sống lại, nhưng người dùng vẫn phải chịu thời gian downtime thêm một lúc nữa trong lúc chờ 30 giây để cả 3 container khởi động lại từ đầu. Đáng lẽ nếu dùng đúng `/readyz`, orchestrator chỉ tạm ngừng gửi traffic mới vào chứ không kill process.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi tôi gặp là **"Invalid value for '--port': '$PORT' is not a valid integer"** khiến Uvicorn crash ngay khi khởi động và Railway báo lỗi "Healthcheck failed".
- **Tìm nguyên nhân**: Xem log deploy của Railway (dùng CLI `railway logs`), tôi phát hiện biến `$PORT` không được biến đổi thành giá trị thật. Nguyên nhân gốc rễ là file `railway.toml` định nghĩa cứng `startCommand` gọi `uvicorn` qua shell nhưng lại không được Railway thực thi đúng cách qua shell expansion. Lệnh đó đè lên `CMD` của Dockerfile.
- **Cách sửa**: Tôi xoá bỏ `startCommand` khỏi file `railway.toml`, đồng thời đổi `CMD` trong Dockerfile thành dạng `CMD ["python", "-m", "app.main"]`. Với cách này, Python khởi động và `pydantic-settings` tự động đọc biến `$PORT` để gán cổng cho Uvicorn cực kì chuẩn xác mà không phụ thuộc vào bash/shell của OS/Cloud.
