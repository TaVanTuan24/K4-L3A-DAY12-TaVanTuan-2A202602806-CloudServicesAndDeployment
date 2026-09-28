# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: viết câu trả lời của bạn ngay bên dưới mỗi câu hỏi.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Tạ Văn Tuấn  Mã học viên: 2A202602806

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Giả sử lần đầu set service lên Railway tôi quên tạo biến AGENT_API_KEY trong dashboard. Nhờ trường `agent_api_key` không có giá trị mặc định, tại lúc import `Settings()` pydantic ném `ValidationError` ngay → process không khởi động được → healthcheck của Railway đỏ → Railway giữ bản deploy cũ đang chạy ổn và in rõ lỗi "thiếu AGENT_API_KEY" trong build/runtime log, nên tôi phát hiện ngay khi vừa triển khai. Ngược lại, nếu để mặc định `"changeme"`, app vẫn start xanh, `/ask` vẫn nhận request — và bất kỳ ai đoán ra chuỗi `"changeme"` đều gọi được API bằng đúng key chung đó; tôi chỉ biết cuối tháng khi hóa đơn tiền LLM tăng đột biến. "Chết sớm" ở đây biến một lỗi cấu hình thành lỗi deploy nhìn thấy ngay, thay vì một lỗ hổng bảo mật/khoản chi chạy ngầm nhiều tuần.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Một dòng log JSON thật, lấy từ log của service Railway production khi tôi gọi `/ask` với `X-User-Id: sv-test`:

```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:48:34.672518+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 37, "cost_usd": 2.265e-05}
```

Hai việc làm được mà `print("đã trả lời xong")` không làm được: (1) máy có thể parse từng trường — tôi lọc `event == "ask_completed"` rồi cộng dồn `cost_usd` theo `user_id` để biết chính xác từng user đã tiêu bao nhiêu; (2) tạo cảnh báo tự động theo ngưỡng (cảnh báo khi tổng `cost_usd` vượt ngân sách) vì giá trị nằm trong trường có tên, không phải chuỗi text tự do phải grep thủ công.

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
| 1 stage (bản đầu) | 1.73 GB |
| Multi-stage | 280 MB |

Giải thích: phần chênh lệch (~1.45 GB) chủ yếu là toàn bộ base image `python:3.11` bản "đầy đủ" — gồm trình biên dịch C/C++, các header phát triển, `git`, man pages và hàng loạt công cụ build không cần thiết lúc chạy. Bản multi-stage chạy trên `python:3.11-slim` (chỉ có runtime tối thiểu) và chỉ mang theo các gói Python đã cài từ stage builder, nên loại bỏ gần hết phần "phục vụ compile" đó.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Trong Dockerfile của tôi, khi sửa 1 ký tự trong `app/main.py`: các layer được DÙNG LẠI từ cache gồm `FROM python:3.11-slim`, `ENV ...`, `COPY requirements.txt` và `RUN pip install -r requirements.txt` (đứng trước mọi lệnh copy source). Các layer phải CHẠY LẠI bắt đầu từ `COPY app ./app` trở xuống (và các bước ngay sau nó như `COPY utils`, `HEALTHCHECK`, `USER`, `CMD`). Nếu đặt `COPY . .` lên TRƯỚC `RUN pip install` thì chỉ cần 1 dòng code đổi, layer `COPY . .` bị invalidate, kéo theo `RUN pip install` chạy lại toàn bộ mỗi lần build — mất vài chục giây và tải lại các package mỗi khi sửa code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Chuỗi sự kiện: (1) code Python có lỗ hổng (ví dụ vô tình thực thi input người dùng hoặc lỗi command injection) → (2) kẻ tấn công khai thác để chạy lệnh tùy ý với đúng quyền của process Python → (3) nếu container chạy bằng root, quyền đó là root của cả namespace container → (4) nhờ kernel dùng chung, root trong container có thể mount filesystem của host, truy cập `/proc`, ổ đĩa hoặc socket Docker và leo thang sang host. Lệnh `USER app` cắt đứt ở bước (3): dù code bị khai thác, kẻ tấn công chỉ có quyền của user `app` không đặc quyền — không ghi được `/etc`, `/usr`, không mount, không leo lên root host, nên phạm vi tổn thất bị giới hạn.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Tối đa 20 request trong 2 giây liên tiếp. Cách đạt: gửi 10 request vào lúc 10:00:59 (phút cũ), và 10 request nữa vào 10:01:00–10:01:01 (phút mới). Vì bộ đếm "phút đồng hồ" reset về 0 tại đúng giây 00, 10 request ở giây 59 thuộc phút cũ và 10 request ở giây 00–01 thuộc phút mới — mỗi phút đều "hợp lệ" 10 request, nhưng thực tế cùng một người đã gửi 20 request trong ~2 giây. Cửa sổ trượt tránh được lỗ này vì nó luôn đếm trong 60 giây GẦN NHẤT chạy liên tục.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn SỐ LƯỢNG request trong một khoảng thời gian (60 giây). Cost guard giới hạn SỐ TIỀN đã chi trong tháng, độc lập với số request. Tình huống rate limit cho qua nhưng cost guard phải chặn: user gửi 1 request/giây (chưa chạm trần 10/phút) nhưng mỗi request nhét câu hỏi dài 50k token → họ đốt hết ngân sách tháng chỉ sau vài request, dù chưa bao giờ bị 429. Tình huống ngược lại: user spam 20 request/phút, mỗi request chỉ vài token (gần như miễn phí) → rate limit chặn ở 429 để bảo vệ hệ thống, trong khi cost guard thấy tổng chi phí vẫn cực nhỏ nên cho qua.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu gộp liveness và readiness thành một và cho nó kiểm tra Redis, khi Redis mất kết nối 30 giây: (1) cả 3 container cùng trả 503 ở probe duy nhất; (2) orchestrator hiểu nhầm là "process chết" (thay vì chỉ "chưa sẵn sàng nhận traffic"); (3) nó restart cả 3 container, rồi container mới lên vẫn không tìm thấy Redis nên lại 503 → lại restart; (4) kết quả là vòng lặp restart kéo sập toàn bộ service, trong khi lẽ ra chỉ cần chờ Redis quay lại. Tách riêng: `/health` vẫn 200 (process sống, không restart vô nghĩa), `/ready` trả 503 để load balancer tạm ngừng đẩy traffic vào instance không phục vụ được.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Tôi đã deploy service lên Railway (Redis nội bộ, 1 replica) và gọi `/ask` 3 lần liên tiếp với cùng `X-User-Id: history-test`. Kết quả `history_length` thật thu được lần lượt là **0 → 2 → 4**: mỗi câu hỏi mới đều nhìn thấy đủ 2 message của lượt trước (user + assistant). Điều này khớp với cách `ConversationStore` cài đặt — history ghi vào Redis List `history:<user_id>` (không có dict/list toàn cục nào trong process, đã được test `test_khong_co_bien_toan_cuc_giu_state` kiểm chứng). Nếu thay bằng một dict Python nội bộ của mỗi process, thì khi scale ngang ra nhiều instance, mỗi instance giữ RAM riêng: request rơi vào container B trong khi lịch sử nằm ở container A → `history_length` sẽ thường xuyên quay về 0 hoặc không tăng (agent "mất trí nhớ" ngẫu nhiên). Với plan hiện tại Railway free chỉ có 1 replica nên tôi chưa chạy thử `--scale agent=3` trên cloud; có thể kiểm tra thêm bằng local Docker với `docker compose up --scale agent=3` + nginx LB theo LAB_GUIDE.md.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Lỗi thật tôi gặp khi triển khai lên Railway: **service đã chạy nhưng chưa có Redis — biến `REDIS_URL` chưa được đặt, và project không có Redis service nào.** Vì `Settings.redis_url` mặc định là `redis://localhost:6379/0`, mà trong container `localhost` chính là container của agent (không có Redis chạy ở đó), nên `/ready` sẽ trả 503 "not ready". Cách phát hiện: chạy `railway variable list -s production-ai-agent-day12` → danh sách biến không có `REDIS_URL`; `railway service list` → project chỉ có đúng 1 service (agent), không có Redis. Cách sửa: (1) chạy `railway add -d redis` để tạo Redis nội bộ Railway; (2) set biến `REDIS_URL=${{Redis.REDIS_URL}}` (dùng variable reference để không hardcode password vào repo/config); (3) redeploy bằng `railway up`. Kết quả sau khi sửa: `/ready` trả 200 `{"status":"ready","redis":true}` — chứng tỏ agent đã kết nối được Redis thật.
