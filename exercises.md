# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder dưới mỗi câu bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Nguyễn Quốc Bảo  Mã học viên: 2A202603011

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Ví dụ: mình deploy lên Render nhưng quên nhập `AGENT_API_KEY` trong dashboard.
> Nếu code có mặc định `"changeme"`, service vẫn lên xanh, `/health` 200, và ai
> đọc repo công khai (thấy chữ `changeme`) đều gọi được `/ask` bằng khóa đó —
> mình chỉ phát hiện khi hóa đơn LLM tăng. Vì không có mặc định, `Settings()`
> ném `ValidationError: agent_api_key Field required` ngay lúc khởi động, deploy
> fail đỏ trên dashboard trong lúc mình còn đang nhìn, và sửa chỉ mất 1 phút.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật khi chạy `docker compose` và gọi `/ask`:
> `{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:13:17.294939+00:00", "user_id": "sv-rl", "tokens_in": 392, "tokens_out": 43, "cost_usd": 8.46e-05}`
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
> 1. Lọc/tổng hợp theo trường: cộng `cost_usd` theo `user_id` để biết user nào
>    tiêu nhiều tiền nhất hôm nay, hoặc đếm số `ask_completed` mỗi phút.
> 2. Đặt cảnh báo tự động trên nền tảng log: ví dụ `level == "error"` tăng đột
>    biến trong 5 phút, hoặc `tokens_in` vượt ngưỡng → gửi alert. Mỗi log là
>    một dòng JSON nên máy parse được ngay, không phải regex chuỗi tự do.

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
| 1 stage (bản đầu) | 1.84 GB (~1840 MB) |
| Multi-stage | 271 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Chênh lệch ~1.57 GB gồm: (1) base image `python:3.11` bản đầy đủ mang theo
> gcc, build-essential, header dev, git, v.v. (~1 GB) mà app chỉ cần lúc cài
> thư viện; bản multi-stage dùng `python:3.11-slim` và bỏ lại stage `builder`;
> (2) cache của pip (bản cũ không có `--no-cache-dir`); (3) bản 1-stage `COPY . .`
> không có `.dockerignore` đầy đủ nên chép cả `.venv`, `.git`, `tests`,
> `__pycache__` — và cả file `.env` chứa secret — vào image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Mình thêm một ký tự vào `app/main.py` rồi `docker build` lại, output cho thấy
> các bước `WORKDIR`, `COPY requirements.txt`, `RUN pip install ...` và
> `COPY --from=builder /install /usr/local` đều `CACHED`; chỉ `COPY app ./app`,
> `COPY utils ./utils` và `RUN useradd` (các layer phía sau) chạy lại — tổng
> build ~1.4 giây. Nếu đặt `COPY . .` trước `RUN pip install` thì layer COPY
> đổi hash mỗi lần sửa code, mọi layer sau nó mất cache → pip cài lại toàn bộ
> thư viện, mỗi lần build tốn vài phút thay vì vài giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: code Python có lỗ hổng (ví dụ deserialize/`eval` dữ liệu người
> dùng, hoặc thư viện bị RCE) → kẻ tấn công chạy lệnh tùy ý trong container với
> quyền của process → nếu process là root (UID 0), nó đọc/sửa được mọi file
> trong container, và UID 0 trong container chính là UID 0 trên kernel host; chỉ
> cần thêm một cấu hình sai (mount `/var/run/docker.sock`, volume thư mục host,
> `--privileged`) hoặc một lỗ hổng kernel/runtime là thoát ra host với quyền root.
> `USER appuser` (UID 10001) cắt chuỗi ở bước 3: shell của kẻ tấn công chỉ là
> user thường, không ghi được file hệ thống, không cài được gói, và nếu có thoát
> ra host thì cũng chỉ là một UID không có quyền gì.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request** trong 2 giây. Cách đạt: gửi 10 request lúc 10:00:59
> (hết hạn mức của phút 10:00), đồng hồ sang 10:01:00 bộ đếm reset về 0, gửi
> tiếp 10 request lúc 10:01:00–10:01:01. Cả 20 request đều "hợp lệ" với cách
> đếm theo phút đồng hồ. Sliding window luôn nhìn 60 giây gần nhất nên ở giây
> 10:01:01 nó vẫn thấy 10 request cũ → request thứ 11 bị 429 (mình kiểm tra
> thực tế: 15 lần gọi liên tiếp → 10 lần 200, 5 lần 429).

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất** (số request/60 giây, trả 429, tự hết sau
> 1 phút); cost guard giới hạn **tổng tiền** trong tháng (trả 402, chỉ reset khi
> sang tháng mới).
> - Rate limit cho qua nhưng cost guard chặn: user gửi đều 5 request/phút (dưới
>   hạn mức 10) nhưng mỗi câu hỏi và lịch sử rất dài, cả ngày liên tục → tổng
>   chi phí tháng vượt 10 USD → 402.
> - Cost guard cho qua nhưng rate limit chặn: user mới, gần như chưa tiêu đồng
>   nào, nhưng một script lỗi gửi 15 request "test" trong một giây → từ request
>   thứ 11 bị 429 dù ngân sách còn gần nguyên.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> 1. Redis mất kết nối → 2. endpoint gộp của cả 3 container cùng trả 503 vì
> đều dựa vào Redis → 3. orchestrator coi đó là liveness fail, sau vài lần
> retry thì **restart cả 3 container** cùng lúc → 4. trong lúc khởi động lại
> không container nào phục vụ, request đang xử lý bị cắt → 5. Redis quay lại
> sau 30 giây nhưng các container có thể vẫn đang restart/lặp restart vì probe
> vẫn fail lúc khởi động → sự cố 30 giây của Redis thành downtime toàn hệ thống.
> Tách ra thì `/health` vẫn 200 (process ổn, không restart), chỉ `/ready` 503 →
> LB tạm ngừng gửi traffic, Redis về là `/ready` 200 và phục vụ tiếp ngay.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Mình chạy `docker compose up -d --scale agent=3` (3 container ở cổng
> 8000/8001/8002) và gọi `/ask` lần lượt vào 8000 → 8001 → 8002 → 8000 → 8001
> với cùng `X-User-Id: sv-scale`: `history_length` = 0, 2, 4, 6, 8 — tăng đều dù
> mỗi lần vào một container khác, vì cả 3 cùng đọc/ghi `history:sv-scale` trong
> Redis. Nếu lưu trong dict Python, mỗi container có dict riêng: lần 1 (8000)
> ra 0, lần 2 (8001) ra 0, lần 3 (8002) ra 0, lần 4 (8000) ra 2, lần 5 (8001) ra
> 2 — con số nhảy lung tung theo container và agent "quên" hội thoại; restart
> container cũng mất sạch.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi mình gặp: sau khi deploy lên Render, `/health` và `/ready` đều 200 nhưng
> gọi `/ask` kèm `X-API-Key` vẫn nhận `401 {"detail":"invalid or missing API key"}`
> (cả 15 lần gọi đều 401, không lần nào ra 429).
> Cách tìm nguyên nhân: `/ask` không có key cũng 401 → auth đang chạy đúng, vấn đề
> nằm ở giá trị khóa. Mình kiểm tra dòng `DEPLOY_API_KEY` trong `.env` (dài 43 ký
> tự, không có dấu cách hay `` thừa) → khóa ở máy sạch, suy ra `AGENT_API_KEY`
> nhập trên dashboard Render khác khóa mình đang gửi. Lần deploy "Live" lúc đó là
> do deploy hook của CI kích hoạt, vẫn dùng biến môi trường cũ.
> Cách sửa: vào Render → Environment, dán lại đúng giá trị khóa (không kèm tên biến,
> không khoảng trắng), "Save, rebuild and deploy", chờ lần deploy mới Live → gọi lại
> thì ra 200, và 15 lần liên tiếp cho 10 lần 200 + 5 lần 429.
> Trước đó mình còn gặp: Railway báo "Your trial has expired" nên chuyển sang
> Render, và bước CI `curl -fsS -X POST "$RENDER_DEPLOY_HOOK_URL"` lỗi
> `curl: (3) URL rejected: Malformed input` vì workflow chạy trước khi tạo secret
> (biến rỗng) → thêm secret rồi Re-run là xanh.
