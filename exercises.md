# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng mẫu bên dưới mỗi câu bằng câu trả lời thực tế.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Trần Vũ Gia Huy  Mã học viên: 2A202602705

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu tôi quên đặt `AGENT_API_KEY` trên Railway, fail fast làm container dừng ngay và lỗi cấu hình xuất hiện trong log lúc tôi vẫn đang theo dõi deployment. Nếu có khóa mặc định `"changeme"`, service vẫn báo online và người ngoài có thể đoán khóa để gọi `/ask`; tôi chỉ phát hiện sau khi quota hoặc ngân sách đã bị tiêu.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi quan sát được là `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T03:05:25.997765+00:00","user_id":"cp4-scale-test","tokens_in":109,"tokens_out":47,"cost_usd":0.00004455}`. Từ log này tôi có thể lọc hoặc đếm request theo `user_id`, đồng thời cộng `cost_usd` và token để theo dõi chi phí. Chuỗi `print("đã trả lời xong")` không có các trường ổn định để máy truy vấn hay tổng hợp.

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
| Multi-stage | 297 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build lại cả hai bản và đo được image một stage là 1.73 GB, còn bản multi-stage là 297 MB, giảm khoảng 1.43 GB. Phần chênh lệch chủ yếu đến từ base image Python đầy đủ và các thành phần chỉ cần khi cài/build dependency; runtime slim chỉ nhận thư viện đã cài cùng source cần chạy nên không mang toàn bộ môi trường build sang production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, các layer base image, `COPY requirements.txt`, `pip install` và copy dependency từ builder được dùng lại từ cache; layer `COPY app ./app` và các layer runtime phía sau nó phải tạo lại. Nếu đặt `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm mất cache của bước cài dependency, khiến build lại chậm dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Một lỗ hổng Python có thể cho kẻ tấn công thực thi lệnh trong container. Nếu process chạy root, lệnh đó có quyền root trong container; kết hợp cấu hình mount hoặc lỗi runtime/container escape, phạm vi ảnh hưởng có thể lan tới dữ liệu và host. `USER appuser` làm process chỉ có UID 10001, nên ngay từ bước thực thi lệnh kẻ tấn công đã bị giới hạn quyền thay vì có root. Tôi đã kiểm tra bằng `id` và thấy `uid=10001(appuser)`.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với bộ đếm theo phút đồng hồ, user có thể gửi 10 request ở giây `xx:59` rồi thêm 10 request ngay sau khi đồng hồ sang phút mới ở giây `xx:00`, tức tối đa 20 request trong khoảng 2 giây. Sliding window 60 giây vẫn nhìn thấy 10 request cũ khi 10 request mới tới nên không để lọt burst này.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tần suất request trong một cửa sổ thời gian, còn cost guard giới hạn tổng tiền theo user trong tháng. Một user gọi ít hơn 10 lần/phút nhưng mỗi request có prompt rất lớn vẫn có thể được rate limit cho qua và bị cost guard chặn. Ngược lại, user còn nguyên ngân sách nhưng gửi request thứ 11 trong 60 giây sẽ bị rate limit trả 429 dù cost guard vẫn cho phép. Khi chạy thật tôi cũng xác nhận trường hợp vượt ngân sách trả 402.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu `/health` cũng kiểm tra Redis, khi Redis mất kết nối thì cả ba container cùng trả 503. Orchestrator coi cả ba process bị lỗi và restart chúng gần như đồng thời, làm mất toàn bộ năng lực phục vụ dù code agent vẫn sống; Redis quay lại nhưng các container còn đang khởi động. Khi thử thực tế, tôi dừng Redis và thấy `/health` vẫn 200 còn `/ready` trả 503; cách tách này chỉ khiến load balancer ngừng gửi traffic mà không restart agent.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi scale ba agent trên các cổng 8000–8002 và gọi cùng `X-User-Id` qua từng replica. `history_length` lần lượt là `0`, `2`, `4`, chứng tỏ các container cùng đọc Redis. Nếu dùng dict Python, mỗi replica có bộ nhớ riêng nên khi request đổi container tôi sẽ thấy lịch sử quay lại 0 hoặc tăng không đều, và lịch sử cũng mất khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi Railway báo deployment `SUCCESS`, domain vẫn trả `502 Application failed to respond`. Tôi đọc `railway logs` và thấy Uvicorn đang nghe `0.0.0.0:8080`, rồi kiểm tra domain thấy nó được tạo với `targetPort: 8000`. Tôi cập nhật domain sang port 8080; sau đó `/health` và `/ready` đều trả 200, request thiếu key trả 401 và request có key trả 200.
