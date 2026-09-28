# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng giữ chỗ (placeholder) dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Nguyễn Việt Dũng  Mã học viên: 2A202602533

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Khi tạo Blueprint trên Render, tôi quên điền `AGENT_API_KEY`.
>
> - **Có mặc định `"changeme"`:** app vẫn chạy, Render báo Live. Nhưng ai đọc code
>   trên GitHub cũng biết khóa `changeme`, nên người lạ gọi `/ask` bằng tiền của tôi.
>   Tôi chỉ phát hiện khi nhìn hóa đơn.
> - **Không có mặc định:** app ném `ValidationError` ngay khi khởi động, deploy báo
>   Failed, log ghi rõ thiếu `agent_api_key`. Tôi sửa được ngay trong vài phút.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:58:59.161550+00:00", "user_id": "sv01", "tokens_in": 423, "tokens_out": 51, "cost_usd": 9.405e-05}
> ```
>
> 1. **Thống kê theo trường:** lọc `event == "ask_completed"`, nhóm theo `user_id`,
>    cộng `cost_usd` để biết user nào tiêu nhiều tiền nhất.
> 2. **Đặt cảnh báo tự động:** ví dụ báo động khi số log `level == "error"` trong
>    5 phút vượt ngưỡng, hoặc khi `tokens_in` quá lớn. Tôi thấy `tokens_in` của
>    `sv01` tăng dần 179 → 423 qua 6 lượt, tức chi phí tăng theo độ dài hội thoại.

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
| 1 stage (bản đầu) | chưa đo được, ước tính ~1.1 GB |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Bản 1 stage tôi build không xong vì mạng chậm (hơn 12 phút chưa tải xong base
> image). Theo Docker Hub, bản nén của `python:3.11` là 409 MB, còn
> `python:3.11-slim` chỉ 45.6 MB.
>
> Phần chênh lệch gồm:
>
> 1. **Base image đầy đủ:** `python:3.11` mang theo compiler (`gcc`, `make`), header
>    phát triển, `git`... chỉ cần lúc build, không cần lúc chạy.
> 2. **Cache của pip:** bản đầu không có `--no-cache-dir`. Bản mới cài ở stage
>    `builder` rồi chỉ copy kết quả sang.
> 3. **File thừa do `COPY . .`:** `.venv`, `tests/`, `.pytest_cache`, thậm chí
>    `.env`. Bản mới chỉ copy `app` và `utils`.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> - **Dùng lại cache:** `COPY requirements.txt`, `RUN pip install`,
>   `COPY --from=builder`, `useradd` (log build báo `CACHED`).
> - **Chạy lại:** `COPY app` và `COPY utils`. Cả lần build chỉ mất vài giây.
>
> Nếu đặt `COPY . .` trước `pip install`, sửa một ký tự code cũng làm `pip install`
> chạy lại từ đầu. Với mạng của tôi, bước đó mất 617.8 giây thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> 1. Code có lỗ hổng cho phép chạy lệnh tuỳ ý.
> 2. Kẻ tấn công có shell trong container với quyền root.
> 3. Root đọc được secret, sửa code, cài công cụ.
> 4. Gặp thêm cấu hình sai (mount `docker.sock`, `--privileged`) hoặc lỗi kernel,
>    kẻ tấn công thoát ra host với quyền root.
>
> `USER appuser` cắt ở **bước 2**: shell chỉ là user thường, không sửa được file hệ
> thống. Nếu thoát ra host cũng không có đặc quyền.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> **20 request.** Gửi 10 request lúc 10:00:59 (phút cũ), bộ đếm reset lúc 10:01:00,
> gửi tiếp 10 request (phút mới). Cả hai phút đều "đúng luật".
>
> Sliding window luôn nhìn lại 60 giây trước mỗi request, nên request thứ 11 bị
> chặn. Tôi thử thật: 12 lần gọi thì 10 lần 200, 2 lần cuối 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit đếm **số request** trong 60 giây (429). Cost guard đếm **số tiền** trong
> tháng (402).
>
> - **Rate limit cho qua, cost guard chặn:** user gửi 5 request/phút nhưng mỗi câu
>   hỏi rất dài, tốn nhiều token. Không vượt hạn mức nhưng tiền vẫn vượt ngân sách.
> - **Cost guard cho qua, rate limit chặn:** một script lỗi gọi 200 lần/phút với câu
>   hỏi ngắn. Tổng chưa tới 0.02 USD nhưng làm quá tải service.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối, cả 3 container cùng trả 503 vì dùng chung Redis.
> 2. Orchestrator tưởng cả 3 process đã chết, nên restart cả 3 cùng lúc.
> 3. Redis quay lại nhưng không còn container nào phục vụ, user nhận lỗi 502/503.
>
> Sự cố nhỏ ở Redis thành sự cố toàn hệ thống. Tách riêng thì `/health` vẫn 200
> (không restart), chỉ `/ready` trả 503 để tạm ngừng nhận traffic. Tôi thử
> `docker compose stop redis`: `/ready` trả 503, `/health` vẫn 200.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, `history_length` tăng đều 0, 2, 4, …, 18 vì mọi container đọc cùng
> một lịch sử. (Tôi đo trên 1 container, vì `--scale agent=3` bị trùng cổng 8000.)
>
> Với dict Python, mỗi container chỉ nhớ lượt của chính nó. Chia vòng tròn A → B → C
> thì con số thành 0, 0, 0, 2, 2, 2…: nhảy lung tung, agent "mất trí nhớ", và mất
> sạch khi container restart.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> **Lỗi:** build image thất bại ở bước `pip install`:
>
> ```
> ReadTimeoutError: HTTPSConnectionPool(host='files.pythonhosted.org', port=443): Read timed out.
> ```
>
> **Nguyên nhân:** Docker chỉ ra lỗi ở `Dockerfile:29` (`pip install`). Traceback
> cho thấy pip tải từ PyPI quá 15 giây nên bỏ cuộc, tức lỗi mạng chứ không phải lỗi
> code.
>
> **Cách sửa:** thêm `--default-timeout=120 --retries=10` cho `pip install`, và bổ
> sung `.dockerignore` để không gửi `.venv` vào build. Sau đó build thành công, và
> deploy lên Render chạy ngay lần đầu.
