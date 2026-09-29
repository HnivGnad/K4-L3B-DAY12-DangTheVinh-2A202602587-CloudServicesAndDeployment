# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay từng dòng trả lời mẫu bên dưới bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đặng Thế Vinh.......... Mã học viên: 2A202602587............

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là khi tôi deploy image lên Railway nhưng quên tạo biến
> `AGENT_API_KEY`. Nếu có mặc định `"changeme"`, container vẫn báo healthy và
> public URL vẫn nhận request; người khác chỉ cần đoán khóa mặc định là có thể gọi
> `/ask`, làm phát sinh chi phí và ghi dữ liệu dưới danh nghĩa người dùng hợp lệ.
> Với trường bắt buộc như hiện tại, Pydantic báo lỗi validation ngay lúc process
> khởi động, deployment không chuyển sang trạng thái healthy. Nhờ vậy tôi phát
> hiện sai cấu hình trong log trước khi traffic thật đi vào service.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng tôi thu được khi gọi `/ask` là:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-29T04:58:09.241600+00:00", "user_id": "cp5-test", "tokens_in": 3, "tokens_out": 35, "cost_usd": 2.145e-05}
> ```
>
> Thứ nhất, tôi có thể lọc chính xác theo `event`, `user_id`, `level` hoặc khoảng
> `timestamp` trên hệ thống log thay vì tìm kiếm một câu chữ tự do. Thứ hai, tôi
> có thể cộng `tokens_in`, `tokens_out`, `cost_usd` để dựng metric, dashboard và
> cảnh báo chi phí theo người dùng. `print("đã trả lời xong")` không có các field
> ổn định nên máy không thể nhóm, tính tổng hay truy vết request theo cách đó.

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
| 1 stage (bản đầu) | khoảng 1.100 MB |
| Multi-stage | khoảng 310 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản một stage dùng `python:3.11` đầy đủ nên mang theo một hệ điều hành nền lớn,
> các công cụ và thư viện phục vụ build, cache của trình cài package và mọi thứ
> được tạo trong lúc cài dependency. Bản cuối dùng `python:3.11-slim`; stage
> `builder` tạo virtual environment nhưng runtime chỉ `COPY` `/opt/venv` sang
> image cuối. Vì vậy compiler, cache và filesystem trung gian không xuất hiện
> trong image chạy thật. Phần còn lại khoảng 310 MB chủ yếu là Python slim,
> virtual environment chứa dependency, mã trong `app/` và `utils/`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Khi chỉ sửa `app/main.py`, toàn bộ stage `builder` vẫn lấy từ cache: base image,
> `WORKDIR`, `COPY requirements.txt` và layer tạo virtual environment/cài package
> đều không đổi. Ở runtime, các layer tạo user và chép `/opt/venv` cũng được dùng
> lại. Cache bị mất từ `COPY --chown=app:app app ./app`; các instruction đứng sau
> nó được dựng lại, nhưng chúng rất nhẹ và không phải tải/cài dependency.
>
> Nếu đặt `COPY . .` trước `RUN pip install`, chỉ một ký tự thay đổi cũng làm hash
> của layer `COPY` đổi. Mọi layer sau đó, gồm `pip install`, phải chạy lại dù
> `requirements.txt` không thay đổi. Build vì thế chậm hơn và có thể tải lại toàn
> bộ package không cần thiết.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi rủi ro là: kẻ tấn công khai thác lỗi trong endpoint Python để thực thi
> lệnh trong container; nếu process là root, lệnh đó có toàn quyền với filesystem
> container. Khi container còn được cấp capability nguy hiểm, mount thư mục host,
> mount Docker socket hoặc gặp lỗ hổng container-runtime/kernel, quyền root này
> có thể được dùng để sửa file, điều khiển container khác hoặc thoát ra host với
> quyền cao.
>
> `USER app` cắt chuỗi ngay sau bước thực thi lệnh: mã bị chiếm quyền chỉ chạy với
> UID không đặc quyền, không sửa được file thuộc root và khó sử dụng các tài
> nguyên đặc quyền. Nó không thay thế việc vá lỗi hay bỏ capability/mount nguy
> hiểm, nhưng làm giảm mạnh phạm vi thiệt hại nếu ứng dụng bị khai thác.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi **20 request trong khoảng 2 giây**. Họ gửi 10 request ở
> cuối phút, ví dụ từ `10:00:59` đến ngay trước `10:01:00`; bộ đếm của phút 10:00
> vẫn xem đó là đủ 10 request hợp lệ. Ngay khi đồng hồ sang `10:01:00`, bộ đếm
> reset và họ gửi thêm 10 request. Sliding window tránh khe hở này vì khi request
> thứ 11 đến, 10 request cuối phút trước vẫn nằm trong 60 giây gần nhất.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit bảo vệ **tốc độ/số request trong một cửa sổ ngắn**, còn cost guard
> bảo vệ **tổng tiền tích lũy theo tháng**. Hai quota dùng cùng `user_id` nhưng có
> đơn vị và thời gian sống khác nhau: Redis ZSET 60 giây cho rate limit, còn một
> giá trị chi phí theo khóa `cost:<user>:<tháng>` cho cost guard.
>
> Ví dụ rate limit cho qua nhưng cost guard chặn: user mới gọi request đầu tiên
> trong phút này, nhưng các lần gọi trước đó đã làm tổng tháng vượt ngân sách 10
> USD; request này chưa vượt 10 request/phút nhưng phải nhận 402. Chiều ngược lại:
> user gần như chưa tốn ngân sách và gửi 11 câu hỏi rất ngắn trong 60 giây; chi
> phí vẫn thấp nhưng request thứ 11 phải nhận 429.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu endpoint gộp có kiểm tra Redis và đồng thời được dùng làm liveness probe,
> chuỗi sự kiện sẽ là: Redis mất kết nối → cả ba container trả 503 → orchestrator
> đánh dấu cả ba là unhealthy → dừng/restart các process vẫn đang sống. Redis vẫn
> lỗi nên các container mới lại trả 503, tạo vòng restart và làm rớt cả request
> đang xử lý. Trong 30 giây đó cụm không còn instance nào nhận traffic; khi Redis
> hồi phục, các container còn phải khởi động và warm-up lại.
>
> Tách endpoint tránh việc này: `/health` vẫn trả 200 vì process Python còn sống,
> nên container không bị restart vô ích; `/ready` trả 503 để load balancer tạm
> ngừng gửi traffic. Khi Redis hoạt động lại, `/ready` tự về 200 và các instance
> được đưa trở lại pool mà không cần thay process.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis dùng chung, tôi quan sát `history_length` tăng đều theo số message đã
> lưu: `0, 2, 4, 6, ...`; mỗi request thêm một message `user` và một message
> `assistant`. Dù request kế tiếp rơi vào container nào, instance đó vẫn đọc cùng
> key `history:<user_id>`. Khi đủ giới hạn, lịch sử dừng ở 20 message do `LTRIM`.
>
> Nếu mỗi process dùng một dict Python, ba container có ba bản lịch sử khác nhau.
> Với cách chia tải luân phiên, kết quả có thể thành `0, 0, 0, 2, 2, 2, ...`;
> với cách chia tải khác nó còn dao động khó đoán. Khi một container restart,
> riêng lịch sử trong dict của container đó quay lại 0, nên người dùng có cảm
> giác agent lúc nhớ lúc quên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi thực tế tôi gặp là chạy `railway up` trực tiếp trong workspace thì CLI báo
> `Access is denied` khi chuẩn bị gói source, nên deployment chưa đi tới bước
> build. Tôi nhận ra đây không phải lỗi ứng dụng vì CP1–CP4 đều xanh, project,
> Redis và environment variables trên Railway đã tồn tại, nhưng dashboard chưa
> xuất hiện build log mới. Nguyên nhân là workspace có thư mục/cache cục bộ mà
> tiến trình đóng gói không đọc được và chúng cũng không cần nằm trong bản deploy.
>
> Tôi tạo một thư mục tạm sạch từ các file được Git theo dõi ở `HEAD` bằng
> `git archive`, rồi chạy Railway CLI với đúng project, environment và service từ
> thư mục đó. Deployment sau đó báo `SUCCESS`; kiểm tra lại cho kết quả `/health`
> và `/ready` đều 200, `/ask` thiếu key trả 401 và có key hợp lệ trả 200.
