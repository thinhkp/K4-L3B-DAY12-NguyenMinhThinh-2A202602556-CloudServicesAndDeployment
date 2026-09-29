# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyen Minh Thinh  Mã học viên: 2A202602556

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Nếu quên cấu hình `AGENT_API_KEY` trên môi trường deploy, app sẽ dừng ngay khi tạo Settings và log lỗi validation. Nhờ vậy mình phát hiện thiếu secret lúc deploy; nếu dùng khóa mặc định `changeme`, service vẫn chạy và người khác có thể dùng khóa đó để gọi API.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Mẫu log theo code: `{"event":"ask_completed","level":"info","timestamp":"<UTC ISO-8601>","user_id":"sv-test","tokens_in":12,"tokens_out":8,"cost_usd":0.0001}`. Đây là mẫu cấu trúc, chưa phải output thu trực tiếp từ lần chạy service của mình. Từ JSON có thể lọc/đếm các request theo `user_id` hoặc `event`, và tổng hợp `cost_usd`/token để theo dõi chi phí; một dòng print tự do không có field ổn định để truy vấn như vậy.

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
| 1 stage (bản đầu) | Chưa đo — không còn Dockerfile một stage trong repo |
| Multi-stage | Chưa đo — Docker build test bị skip trong môi trường hiện tại |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Image nhiều stage chỉ chép dependency đã cài vào runtime, bỏ các file/build artifact chỉ cần ở builder. Vì vậy thường giảm kích thước. Mình chưa có dung lượng đo thực tế vì chưa chạy được Docker build; cần ghi `docker images` sau khi build hai bản để điền số MB và giải thích theo số đo của mình.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, layer cài requirements vẫn được cache vì `requirements.txt` không đổi; các layer copy source và lệnh khởi chạy phía sau sẽ được tạo lại. Nếu `COPY . .` đặt trước `RUN pip install`, mỗi lần sửa source làm layer COPY đổi, kéo theo layer cài dependency chạy lại dù requirements không đổi, khiến build lâu hơn.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu code có lỗ hổng cho phép chạy lệnh tùy ý, kẻ tấn công có thể chiếm quyền trong container. Nếu process là root, họ có quyền cao bên trong container và có thể khai thác cấu hình/kernel hoặc mount Docker socket/volume để tìm đường ảnh hưởng host. `USER app` giảm quyền process ngay từ đầu, nên sau khi khai thác họ chỉ có quyền của user thường; đây là giảm rủi ro chứ không thay thế việc cô lập container và vá lỗi.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Với giới hạn 10 mỗi phút theo phút đồng hồ, có thể gửi 10 request ngay trước khi phút đổi (ví dụ 12:00:59) rồi 10 request ngay sau khi đổi (12:01:00–12:01:01): tối đa 20 request trong khoảng hai giây. Sliding window 60 giây đếm cả hai nhóm trong cùng cửa sổ nên chặn nhóm thứ hai khi đã đủ 10.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượt gọi trong 60 giây; cost guard giới hạn tổng USD của user trong tháng UTC. Ví dụ còn quota request nhưng ngân sách còn 0.001 USD và ước tính lượt mới vượt phần còn lại thì rate limit cho qua còn cost guard trả 402. Ngược lại, nếu user đã gọi đủ 10 lần trong cửa sổ nhưng tháng vẫn còn ngân sách, limiter trả 429 dù cost guard riêng lẻ vẫn cho qua.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối thì cả ba container đều thấy probe gộp thất bại. Load balancer đánh dấu cả ba unhealthy; nếu probe đó bị cấu hình như liveness, orchestrator có thể lần lượt restart cả cụm dù tiến trình còn sống. Trong lúc Redis gián đoạn 30 giây, restart không sửa được dependency nên các container mới vẫn fail probe; traffic bị ngắt và có thể tạo vòng restart. Tách probe giúp `/health` tiếp tục báo process sống còn `/ready` báo không nhận traffic.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, mọi instance đọc cùng danh sách lịch sử nên `history_length` tăng theo số message trước đó bất kể request tới container nào (mỗi lượt hỏi thêm hai message). Nếu lưu trong dict từng process, request luân phiên qua ba container sẽ thấy các history riêng: con số có thể trở về 0/nhỏ khi sang instance khác rồi tăng lại khi quay về instance cũ. Mình chưa chạy scale thực tế vì Docker daemon/build chưa được xác nhận.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Mình chưa deploy lên cloud trong phiên làm bài nên không có lỗi deploy thật để báo cáo. Cần bổ sung câu này sau khi deploy: chép nguyên thông báo từ Railway/Render logs, kiểm tra biến hoặc health check liên quan, rồi ghi nguyên nhân và thay đổi đã khắc phục. Mình không muốn tự dựng một lỗi hay kết quả không xảy ra.
