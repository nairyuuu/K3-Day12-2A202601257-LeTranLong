# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `placeholder` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Trần Long  Mã học viên: 2A202601257

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Khi deploy app lên server production (ví dụ Railway/Render), nếu quên cấu hình biến môi trường `AGENT_API_KEY`, app có giá trị mặc định `"changeme"` vẫn khởi động bình thường. Tuy nhiên, kẻ tấn công hoặc bot quét có thể truy cập API bằng key mặc định `"changeme"` này để gọi LLM trái phép, làm rò rỉ dữ liệu hoặc làm tăng vọt hóa đơn LLM của bạn mà bạn không hề hay biết cho đến khi nhận được hóa đơn thanh toán. Ngược lại, việc "chết sớm" (fail fast) khiến app lỗi ngay lập tức lúc deploy, buộc ta phải cấu hình đúng key trước khi app nhận bất cứ request nào từ Internet.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

JSON log line:
`{"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T02:49:52.415082+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}`

Hai việc có thể làm với dòng log JSON này mà `print()` không làm được:
1. Cấu hình các công cụ gom log (như Datadog, ELK, AWS CloudWatch) tự động phân tích (parse) cú pháp JSON, trích xuất trường `cost_usd` để vẽ biểu đồ theo dõi chi phí LLM theo thời gian thực và tự động gửi cảnh báo (Slack/Email) khi chi phí vượt ngưỡng an toàn.
2. Truy vấn nâng cao, ví dụ: thực hiện thống kê xem `user_id` nào tiêu thụ nhiều token nhất trong ngày bằng cách lọc theo trường `user_id` và tính tổng `tokens_in` + `tokens_out`, giúp dễ dàng phát hiện các hành vi lạm dụng hệ thống.

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản | Dung lượng |
|-----|-----------|
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 143 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Phần dung lượng chênh lệch bao gồm:
1. Sự khác biệt giữa base image gốc `python:3.11` (chứa đầy đủ các compiler, build tools, headers, thư viện phát triển, hệ điều hành Debian đầy đủ) và `python:3.11-slim` (chỉ chứa runtime tối thiểu để chạy Python).
2. Multi-stage build loại bỏ các caching của pip và các build artifacts không cần thiết sinh ra trong quá trình cài đặt thư viện ở builder stage, giữ cho image runtime sạch và nhẹ nhất có thể.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- Với Dockerfile hiện tại: các layer thiết lập môi trường, `COPY requirements.txt .`, và `RUN pip install` được dùng lại hoàn toàn từ cache. Chỉ có layer `COPY . .` và các layer sau đó (EXPOSE, HEALTHCHECK, CMD) phải chạy lại vì file `app/main.py` thay đổi.
- Nếu đặt `COPY . .` lên trước `RUN pip install`: mỗi khi thay đổi bất kỳ ký tự nào trong code, cache của layer `COPY . .` bị mất hiệu lực, dẫn đến layer kế tiếp là `RUN pip install` cũng phải chạy lại hoàn toàn từ đầu (tải và cài lại tất cả các gói thư viện từ Internet). Điều này làm tăng thời gian build rất nhiều lần.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

- Chuỗi sự kiện:
  1. Kẻ tấn công khai thác thành công một lỗ hổng trong code Python (ví dụ: Remote Code Execution qua thư viện lỗi hoặc chức năng upload/run script bừa bãi).
  2. Kẻ tấn công chiếm được quyền shell bên trong container. Do container mặc định chạy bằng root, shell của kẻ tấn công có quyền root bên trong container.
  3. Kẻ tấn công tìm cách thoát khỏi container (container escape) thông qua các lỗ hổng nhân hệ điều hành (kernel exploits), hoặc qua các mount socket/directory không an toàn.
  4. Khi thoát ra máy host thành công, tiến trình của kẻ tấn công kế thừa quyền của tiến trình gốc trong container. Vì tiến trình gốc chạy bằng root trên host, kẻ tấn công nghiễm nhiên sở hữu quyền root cao nhất trên máy host.
- Lệnh `USER appuser` cắt đứt chuỗi này ở bước số 2: Nó hạ quyền của tiến trình chạy app xuống thành một người dùng thường (non-privileged user). Kẻ tấn công chỉ có quyền thường trong container, không thể đọc/ghi các file hệ thống và gặp khó khăn cực lớn khi thực hiện container escape, bảo vệ máy host an toàn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa gửi được 20 request trong 2 giây liên tiếp.
Giải thích: Giả sử giới hạn là 10 request/phút đồng hồ (hệ thống reset bộ đếm vào giây thứ 00 của mỗi phút mới). Người dùng có thể gửi 10 request liên tiếp từ giây 59 của phút trước (ví dụ 10:00:59) và tiếp tục gửi thêm 10 request nữa vào giây 01 của phút sau (10:01:01). Tổng cộng có 20 request được gửi trong khoảng thời gian chỉ 2 giây (từ 10:00:59 đến 10:01:01) mà hệ thống đếm theo phút đồng hồ vẫn coi là hợp lệ.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

- Sự khác nhau: Rate limit giới hạn *tần suất* (số lượng request gọi lên hệ thống trong một đơn vị thời gian), trong khi Cost Guard giới hạn *ngân sách* (số tiền/token tối đa được tiêu dùng trong một tháng).
- Tình huống Rate Limit cho qua nhưng Cost Guard chặn: Một user gửi request đầu tiên trong ngày (Rate Limit ok vì chưa gọi lần nào trong 60s), nhưng request này quá dài hoặc tài khoản của user này đã cạn kiệt ngân sách tháng (Cost Guard chặn vì tổng tiền tiêu dùng vượt Monthly Budget).
- Tình huống Cost Guard cho qua nhưng Rate Limit chặn: User mới đăng ký tài khoản, tài khoản còn nguyên 10$ ngân sách chưa tiêu gì (Cost Guard cho qua), nhưng user dùng tool spam gọi 100 request liên tiếp trong 5 giây (Rate Limit chặn ngay lập tức vì vượt quá 10 request/phút).

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

1. Redis mất kết nối đột ngột trong 30 giây.
2. Orchestrator (Docker/K8s/Railway) định kỳ gọi liveness probe (gộp chung kiểm tra Redis) trên cả 3 container agent.
3. Cả 3 container đều trả về lỗi 500/503 vì Redis chết.
4. Orchestrator kết luận cả 3 container agent này đã "chết cứng" (liveness failed) và lập tức restart (khởi động lại) toàn bộ cả 3 container agent cùng một lúc.
5. Các container agent mới khởi động lên cũng không thể bắt đầu phục vụ (vẫn báo lỗi liveness do Redis vẫn đang mất kết nối), tạo ra vòng lặp restart liên tục (crash loop).
6. Khi Redis kết nối trở lại sau 30 giây, hệ thống vẫn bị gián đoạn (downtime kéo dài thêm) do các container agent đang trong quá trình boot/khởi động lại hoặc bị kẹt trong crash loop.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Nếu lưu lịch sử trong một dict Python trong RAM, khi load balancer phân phối các request ngẫu nhiên đến 3 container agent khác nhau (A, B, C):
- Giá trị `history_length` trong response sẽ nhảy lung tung hoặc tăng giảm không đều (ví dụ: request 1 vào A -> history_length = 0; request 2 vào B -> history_length = 0; request 3 vào A -> history_length = 1; request 4 vào C -> history_length = 0).
- User sẽ thấy hội thoại bị "mất trí nhớ" ngẫu nhiên do mỗi container chỉ lưu giữ một phần lịch sử riêng của nó trong bộ nhớ RAM cục bộ, không thể chia sẻ hay đồng bộ hóa với các container khác.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- Lỗi gặp phải: Health check timeout / Connection refused khi deploy lên cloud platform.
- Thông báo lỗi: Platform logs hiển thị `Health check failed on port 8000: connection refused` hoặc `Service start timeout after 60 seconds`.
- Nguyên nhân: App FastAPI được cấu hình chạy cố định cổng 8000 trong code (`port = 8000` mặc định) hoặc bind vào localhost `127.0.0.1`. Trên cloud (như Railway/Render), hệ thống tự động gán một cổng ngẫu nhiên thông qua biến môi trường `PORT` và load balancer bên ngoài cần bind vào địa chỉ `0.0.0.0` để định tuyến traffic vào container. Do app chạy cổng cứng 8000 và không nghe trên `0.0.0.0`, load balancer không thể truy cập endpoint `/health`.
- Cách sửa: Sử dụng cấu hình `port: int = 8000` kế thừa từ `BaseSettings` của `pydantic-settings` tự động đọc biến môi trường `PORT`, đồng thời chạy uvicorn với tham số `--host 0.0.0.0` và `--port ${PORT:-8000}` trong lệnh khởi chạy CMD của Dockerfile.
