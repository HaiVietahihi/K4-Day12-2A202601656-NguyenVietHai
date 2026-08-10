# Phiếu Phản Ánh — K4 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng hướng dẫn bằng câu trả lời của bạn.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Việt Hải  Mã học viên: 2A202601656

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `api_token` không có giá trị mặc định nên app chết ngay khi
khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà việc
"chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy ứng dụng lên cloud nhưng quên cài đặt biến API_TOKEN. Nếu để mặc định "changeme", ứng dụng vẫn chạy và kẻ xấu có thể dùng token mặc định đó gọi API miễn phí gây tốn chi phí. Việc "chết sớm" giúp hệ thống báo lỗi ngay lúc deploy để sửa kịp thời trước khi nhận traffic.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/chat` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log JSON mẫu: `{"event": "chat_completed", "severity": "INFO", "ts": "2026-08-10T14:50:00+00:00", "client_id": "sv01", "usd_cost": 0.12}`
> 
> Hai việc làm được:
> 1. Lọc và tìm kiếm cấu trúc: Hệ thống (Cloud Watch/Datadog) có thể lọc tự động theo trường như severity="ERROR" hoặc client_id="sv01".
> 2. Thống kê và cảnh báo: Có thể tính tổng usd_cost hoặc đếm số lượng lỗi tự động theo khoảng thời gian để phát cảnh báo.

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
| 1 stage (bản đầu) | 1010 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Phần dung lượng chênh lệch (~740 MB) bao gồm:
> 1. Base Image tối giản: Bản 1-stage dùng python:3.11 chứa toàn bộ trình biên dịch (gcc/g++), công cụ build và package hệ thống thừa. Bản multi-stage chuyển sang dùng python:3.11-slim.
> 2. Loại bỏ công cụ build ở runtime: Stage builder đảm nhận việc biên dịch/cài đặt dependency, stage runtime chỉ copy duy nhất kết quả đã cài đặt, không mang theo bộ công cụ biên dịch hay pip cache.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi sửa 1 ký tự trong app/main.py: Các layer từ FROM, COPY requirements.txt, đến RUN pip install đều được tái sử dụng từ cache. Chỉ layer COPY . . và các bước đứng sau mới phải chạy lại.
> 
> Nếu đặt COPY . . trước RUN pip install: Mọi thay đổi nhỏ trong code đều làm vô hiệu hóa (invalidate) cache của layer COPY . ., buộc Docker phải chạy lại RUN pip install cài đặt lại toàn bộ thư viện từ đầu gây tốn thời gian.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi tấn công khi chạy bằng root: Lỗi lỗ hổng ứng dụng (RCE) ➔ Kẻ tấn công thực thi code với quyền root trong container ➔ Khai thác lỗ hổng Kernel / Container Escape ➔ Chiếm quyền root điều khiển toàn bộ máy host.
> 
> Vị trí lệnh USER cắt đứt chuỗi: Lệnh USER appuser giới hạn tiến trình chạy bằng tài khoản thường (non-root). Khi bị tấn công, kẻ thủ ác bị khóa chặt trong phạm vi quyền hạn thấp của container và không thể leo quyền kiểm soát máy host.

---

### Câu 6 — Bearer token (CP3)

Vì sao 401 phải kèm header `WWW-Authenticate: Bearer`? Và vì sao ta trả **cùng
một** thông báo lỗi cho cả ba trường hợp (thiếu header, sai scheme, sai token)
thay vì nói rõ sai ở đâu cho người dùng dễ sửa?

> Kèm header WWW-Authenticate: Bearer: Tuân thủ chuẩn HTTP (RFC 6750) nhằm chỉ dẫn cho client biết phương thức xác thực mà server yêu cầu là Bearer.
> 
> Trả cùng một thông báo lỗi: Để bảo mật chống rò rỉ thông tin (Information Leakage). Nếu phân biệt "sai scheme" hay "sai token", kẻ tấn công sẽ biết từng bước mình làm đúng hay sai để dò quét (brute-force) hệ thống dễ dàng hơn.

---

### Câu 7 — Token bucket (CP3)

Với `capacity=10`, `refill_per_minute=10`: một client im lặng 10 phút rồi gửi
liên tiếp. Nó gửi được bao nhiêu request trước khi bị 429? Nếu bỏ đoạn
`min(capacity, ...)` trong `available()` thì con số đó thành bao nhiêu, và tại sao?

> Số request gửi được trước khi bị 429: 10 request (vì xô bị giới hạn trần bởi capacity = 10).
> 
> Nếu bỏ min(capacity, ...): Con số đó thành 110 request (10 token ban đầu + 100 token nạp thêm sau 10 phút). Khi đó, client im lặng lâu sẽ tích lũy token vô hạn, làm mất tác dụng chống đợt tấn công dồn dập (burst attack).

---

### Câu 8 — Ngân sách theo ngày (CP3)

So sánh hạn mức $30/tháng với hạn mức $1/ngày cho cùng một client. Giả sử có sự
cố khiến một client gọi liên tục từ 2h sáng. Với mỗi cách, thiệt hại tối đa là
bao nhiêu và service tự hồi phục khi nào?

> Hạn mức $30/tháng: Thiệt hại tối đa là $30 (bị đốt sạch cả ngân sách tháng chỉ trong vài giờ). Service chỉ tự hồi phục khi sang tháng sau (chờ tối đa 30 ngày).
> 
> Hạn mức $1/ngày: Thiệt hại tối đa là $1 (chỉ mất tiền của 1 ngày). Service tự hồi phục vào 00:00 giờ UTC sáng hôm sau mà không cần can thiệp thủ công.

---

### Câu 9 — /healthz khác /readyz (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối: Liveness probe kiểm tra thấy Redis chết ➔ Trả về lỗi 503.
> 2. Orchestrator hiểu nhầm app đã chết: Orchestrator (Docker/Kubernetes) nhận 503 từ Liveness Probe nên cho rằng process ứng dụng bị hỏng.
> 3. Restart toàn bộ cụm container: Orchestrator tiêu diệt (kill) và tiến hành khởi động lại đồng loạt cả 3 container.
> 4. Vòng lặp khởi động lại (CrashLoopBackOff): Các container khởi động lại nhưng Redis vẫn chưa xong ➔ Probe lại thất bại ➔ Tiếp tục bị restart liên tục.
> 5. Hậu quả: Biến sự cố chập chờn nhỏ của Redis (30 giây) thành sự cố sập toàn bộ hệ thống do cả cụm bị restart liên hoàn.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy lên Render, gặp lỗi health check timeout khiến service bị ngắt kết nối. Nguyên nhân: Dockerfile cố định cổng 8000 trong khi Render tự cấp phát biến môi trường PORT khác. Cách tìm nguyên nhân: đọc log trên Render Dashboard thấy uvicorn bind 8000 nhưng healthcheck probe gọi vào $PORT. Cách sửa: cấu hình Uvicorn đọc động từ biến môi trường PORT.
