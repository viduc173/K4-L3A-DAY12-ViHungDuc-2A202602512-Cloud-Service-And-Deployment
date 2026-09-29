# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ dưới mỗi câu hỏi bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Vi Hùng Đức  Mã học viên: 2A202602512

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi deploy lên Render, nếu tôi quên cấu hình `AGENT_API_KEY`, ứng dụng dừng ngay
> lúc khởi động với `ValidationError`. Nhờ vậy tôi phát hiện cấu hình thiếu trong
> log deploy trước khi service nhận traffic. Nếu dùng mặc định `"changeme"`, app
> vẫn chạy và người biết hoặc đoán được khóa mặc định có thể gọi `/ask`, làm phát
> sinh chi phí mà tôi không nhận ra ngay.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một log tôi nhận được có dạng:
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T03:44:42+00:00","user_id":"sv01","tokens_in":1,"tokens_out":35,"cost_usd":0.00002115}`.
> Từ các trường JSON, tôi có thể cộng `cost_usd` theo từng `user_id` để tìm người
> dùng tốn nhiều tiền nhất và đếm số event theo `level` trong một khoảng thời gian
> để tạo cảnh báo. Chuỗi `print("đã trả lời xong")` không có user, timestamp hay
> chi phí nên không làm được hai việc này một cách đáng tin cậy.

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
| 1 stage (bản đầu) | Chưa đo được trên máy local |
| Multi-stage | Chưa đo được trên máy local |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Test cấu trúc CP2 đạt 14/14 nhưng hai test build thật bị skip vì Docker Engine
> local chưa hoạt động, nên tôi không ghi số MB giả. Về thành phần, bản một stage
> dùng image Python đầy đủ và giữ cả môi trường/công cụ phục vụ cài dependency.
> Bản multi-stage dùng `python:3.11-slim`; stage runtime chỉ nhận thư viện đã cài
> từ `/install`, không mang theo phần trung gian của builder, nên image cuối nhỏ
> hơn và có ít bề mặt tấn công hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Với Dockerfile hiện tại, khi chỉ sửa `app/main.py`, các layer chọn base image,
> `COPY requirements.txt` và `RUN pip install` vẫn dùng lại cache vì file dependency
> không đổi. Layer `COPY . .` và các layer đứng sau nó phải chạy lại. Nếu đặt
> `COPY . .` trước `RUN pip install`, mọi thay đổi source đều làm invalid cache từ
> đó trở đi và pip phải cài lại toàn bộ thư viện dù `requirements.txt` không đổi.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Nếu endpoint Python có lỗ hổng cho phép thực thi lệnh, kẻ tấn công trước hết có
> quyền của process trong container. Nếu process là root và có thêm cấu hình nguy
> hiểm như mount socket/thư mục host hoặc capability mạnh, họ có thể sửa file,
> điều khiển Docker hay leo thang sang host. `USER appuser` cắt chuỗi này ở bước
> đầu: mã bị khai thác chỉ chạy với UID 10001 và không có quyền root trong
> container. Nó không loại bỏ lỗ hổng nhưng giảm đáng kể phạm vi thiệt hại.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Có thể gửi tối đa 20 request trong khoảng 2 giây: gửi 10 request ở giây
> `10:00:59`, bộ đếm của phút 10:00 vẫn cho phép; ngay khi sang `10:01:00`, bộ
> đếm reset và người dùng gửi thêm 10 request. Sliding window nhìn lại đúng 60
> giây nên vẫn thấy 10 request cũ và chặn đợt thứ hai.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit kiểm soát tốc độ request trong 60 giây, còn cost guard kiểm soát tổng
> tiền theo user trong cả tháng. Một user gửi đúng một request rất dài có thể vẫn
> qua rate limit nhưng bị cost guard chặn vì chi phí dự kiến làm vượt ngân sách.
> Ngược lại, user còn nhiều ngân sách nhưng gửi 11 request nhỏ liên tiếp trong một
> phút sẽ bị rate limit chặn request thứ 11 dù tổng chi phí vẫn rất thấp.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Redis mất kết nối làm endpoint gộp trả 503; orchestrator hiểu đó là liveness
> failure và lần lượt đánh dấu cả ba container unhealthy. Sau đủ số lần retry, nó
> restart cả ba container dù process ứng dụng không hỏng. Trong lúc Redis phục
> hồi, các container cũng đang khởi động lại nên không còn instance phục vụ và
> sự cố dependency 30 giây biến thành gián đoạn toàn hệ thống. Tách `/ready` giúp
> load balancer chỉ ngừng gửi traffic, còn `/health` vẫn 200 nên container không
> bị restart vô ích.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Trong test CP4 với hai `ConversationStore` dùng chung Redis giả, request đầu có
> `history_length=0` và request thứ hai thấy `history_length=2` (một message user
> và một message assistant). Khi nhiều container dùng Redis thật, con số tiếp tục
> tăng nhất quán dù request vào instance khác. Nếu dùng dict riêng trong từng
> process, số liệu sẽ tăng không đều, có thể quay lại 0 khi request được chuyển
> sang container chưa từng phục vụ user đó, và mất hẳn sau khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Sau khi deploy, lần gọi `/ask` từ PowerShell trả `422 Unprocessable Entity` với
> thông báo `JSON decode error: Unterminated string`. Tôi đọc body lỗi và thấy dữ
> liệu đã bị escape quá nhiều (`{\\"question\\":...}`), đồng thời lệnh sao chép có
> cú pháp Markdown. Tôi chuyển sang `Invoke-RestMethod`, tạo header bằng hashtable
> và gửi body JSON hợp lệ `'{"question":"Hello"}'`. Request sau đó trả 200 với
> `answer`, `user_id`, `history_length`, `cost_usd` và `tokens`.
