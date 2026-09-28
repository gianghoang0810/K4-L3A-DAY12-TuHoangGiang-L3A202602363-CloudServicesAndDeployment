# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Từ Hoàng Giang  Mã học viên: L3A202602363

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

Ví dụ: tạo service Railway mới nhưng quên đặt AGENT_API_KEY. Settings báo lỗi ngay startup, không nhận traffic bằng khóa mẫu. Nếu mặc định changeme, người biết khóa mẫu có thể gọi API. Lifespan chủ động gọi get_settings() nên không đợi request đầu tiên mới phát hiện thiếu cấu hình.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

Log từ HTTP thật tới Uvicorn dùng Redis thật:

```json
{"user_id": "verification-4e73d67c298843d49c76e629b510ee81", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05, "event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:15:50.149260+00:00"}
```

Có thể cộng cost_usd theo user để theo dõi chi phí và đối chiếu tokens_in/tokens_out theo timestamp để tìm lượt gọi tốn tài nguyên. Một dòng print chung chung không có trường để máy lọc và tổng hợp.

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
| 1 stage (bản đầu) | 1,728,348,596 bytes ≈ 1,728.35 MB (1.73 GB) |
| Multi-stage | 309,563,788 bytes ≈ 309.56 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

Tôi đo bằng `docker image inspect` sau khi build thành công cả hai image. Bản
multi-stage nhỏ hơn 1,418,784,808 bytes, tương đương giảm khoảng 82.09%, và
đạt yêu cầu dưới 500 MB. Phần chênh lệch chủ yếu đến từ việc bản một stage dùng
image `python:3.11` đầy đủ, còn bản production dùng `python:3.11-slim`. Bản
multi-stage cũng chỉ đưa virtual environment cùng mã nguồn cần chạy sang stage
runtime; các layer và tệp chỉ phục vụ quá trình build không trở thành nội dung
của image runtime cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

Sau khi tôi chỉ thêm một dấu chấm vào comment trong `app/main.py` rồi build
lại bằng `docker build --progress=plain`, Docker báo `CACHED` cho base image,
`WORKDIR`, `COPY requirements.txt`, bước tạo virtual environment, bước
`pip install`, bước tạo user thường và `COPY --from=builder /opt/venv`.
`COPY app/ ./app/` phải chạy lại vì nội dung trong `app/` đã đổi. Layer
`COPY utils/ ./utils/` cũng được tạo lại vì nó đứng sau layer `COPY app/` đã
thay đổi, rồi Docker export image mới. Build lại chỉ mất vài giây thay vì phải
cài lại toàn bộ thư viện. Nếu đặt `COPY . .` trước `RUN pip install`, bất kỳ
thay đổi nào trong source cũng làm layer `COPY` đổi và vô hiệu toàn bộ cache
phía sau, khiến `pip install` phải chạy lại dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

Một lỗ hổng trong ứng dụng Python có thể cho phép kẻ tấn công thực thi lệnh
trong container. Nếu ứng dụng chạy bằng root, các lệnh đó có quyền root bên
trong container và có thể sửa file hệ thống hoặc khai thác cấu hình nguy hiểm
như bind mount nhạy cảm, Docker socket, chế độ privileged hay lỗ hổng runtime
để tác động tới host. Dockerfile của tôi chuyển sang `USER agent`; kết quả
`docker compose exec agent id` là
`uid=999(agent) gid=999(agent) groups=999(agent)`. Vì vậy, nếu ứng dụng bị
chiếm quyền, mã độc chỉ có quyền của user 999 và bị chặn tại các thao tác cần
root trong container. Biện pháp này làm giảm phạm vi thiệt hại; nó không tự
loại bỏ mọi khả năng container escape, nên vẫn cần tránh mount/privileged và
cập nhật Docker runtime.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

Fixed window cho phép 10 request lúc 10:00:59 và thêm 10 lúc 10:01:00 khi bộ đếm reset, tổng 20 request trong hai giây. Sliding window vẫn tính nhóm đầu trong 60 giây gần nhất nên chặn nhóm tiếp theo.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

Rate limit giới hạn tần suất, cost guard giới hạn tiền theo tháng. User gọi chậm nhưng đã tiêu 11 USD với budget 10 USD thì rate limit cho qua nhưng cost guard trả 402. User chưa hết budget nhưng gửi request thứ 11 trong 60 giây thì limiter trả 429. Kiểm thử cloud thực tế xác nhận rate limit trả 429; `test_cp3.py` xác nhận cost guard trả 402 khi vượt ngân sách. Cost guard của lab không giữ trước ngân sách cho các request đồng thời.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

Nếu probe gọi Redis và được orchestrator dùng làm liveness: Redis mất kết nối -> probe thất bại -> sau ngưỡng lỗi có thể restart cả ba container -> restart app không sửa được Redis -> tiếp tục lỗi tới khi Redis phục hồi. Tách health giữ process sống và ready=503 để ngừng traffic giúp phục hồi mà không restart thừa. Docker HEALTHCHECK một mình không tự restart; Railway healthcheck khi deploy không phải giám sát liên tục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

Tôi chạy ba container `agent` sau Nginx và gọi `/ask` năm lần với cùng một
`X-User-Id`. Log xác nhận request được phân phối tới cả `agent-1`, `agent-2`
và `agent-3`, còn `history_length` trong response vẫn tăng liên tục theo thứ
tự `0, 2, 4, 6, 8`. Mỗi request ghi hai message (`user` và `assistant`) vào
cùng Redis nên container nhận request kế tiếp vẫn đọc được toàn bộ lịch sử.
Nếu thay Redis bằng một dict Python, mỗi container chỉ thấy dữ liệu trong RAM
của chính nó. Khi request luân phiên giữa ba container, kết quả có thể thành
`0, 0, 0, 2, 2, ...` thay vì tăng liên tục; lịch sử còn mất hẳn khi container
restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

Khi mở Public URL `/health`, tôi gặp lỗi `502 Application failed to respond`.

Tôi mở Deploy Logs và thấy ứng dụng đã khởi động thành công, không có traceback:
`Application startup complete` và
`Uvicorn running on http://0.0.0.0:8080`.

Sau đó tôi kiểm tra `Settings → Networking` và thấy domain công khai đang có
Target Port là `8000`. Railway Edge Proxy gửi request tới cổng 8000 trong khi
Uvicorn thực tế lắng nghe ở cổng 8080, nên proxy không kết nối được và trả 502.

Tôi sửa Target Port của public domain từ `8000` thành `8080`. Sau khi lưu thay
đổi, `/health` trả HTTP 200 với `status: ok`, còn `/ready` trả HTTP 200 với
`redis: true`. Lỗi này giúp tôi phân biệt deployment thành công với public
routing hoạt động đúng: container có thể đang chạy nhưng domain vẫn lỗi nếu
Target Port không khớp.
