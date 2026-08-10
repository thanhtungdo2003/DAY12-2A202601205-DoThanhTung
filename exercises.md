# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Thanh Tùng  Mã học viên: 2A202601205

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống của tôi: khi deploy lên Railway, tôi khai báo biến môi trường
> trong dashboard nhưng gõ nhầm tên thành `API_KEY` thay vì `AGENT_API_KEY`.
> Với cách fail fast (`agent_api_key: str` không có mặc định, `app/config.py:47`),
> pydantic-settings ném `ValidationError` ngay lúc khởi động → uvicorn không lên
> → platform báo deploy fail → tôi thấy lỗi trong log lập tức và sửa ngay.
> Nếu để mặc định `"changeme"`: app khởi động bình thường, health check xanh,
> deploy trông "thành công". Nhưng `"changeme"` nằm ngay trong code của repo
> public — ai cũng đọc được. Kẻ lạ chỉ cần gọi `/ask` với
> `X-API-Key: changeme` là dùng agent của tôi miễn phí, và tôi chỉ phát hiện
> khi ngân sách bị đốt sạch hoặc hóa đơn tăng — quá muộn. Fail fast biến lỗi
> cấu hình thành lỗi ngay lúc deploy, thay vì thành lỗi bảo mật lúc vận hành.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một dòng log tôi thu được từ `log_event` (`app/logging_utils.py`):
>
> ```
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:21:14.123456+00:00", "user_id": "sv01", "tokens_in": 9, "tokens_out": 41, "cost_usd": 0.00012}
> ```
>
> Hai việc tôi làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc/đếm theo field bằng máy**: dùng jq hoặc log query của platform
>    (CloudWatch Logs Insights, Datadog...), ví dụ `select(.cost_usd > 0.01)`
>    để tìm request đắt tiền, đếm số request theo từng `user_id`, hay cộng
>    dồn `cost_usd` theo ngày để phát hiện vượt ngân sách. Với print văn bản
>    thì chỉ grep được nguyên dòng, không tách được giá trị, không nhóm được.
> 2. **Cảnh báo tự động dựa trên cấu trúc**: mỗi dòng có `event` cố định và
>    field đặt tên chuẩn, nên hệ thống giám sát đếm được sự kiện/phút và bắn
>    cảnh báo khi `level="error"` hoặc khi `cost_usd` vượt ngưỡng. Một chuỗi
>    in thường thì phải dùng regex mong manh — đổi một từ trong câu thông báo
>    là parser vỡ.

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
| 1 stage (bản đầu) | 1.69 GB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Tôi build cả hai bản trên máy và đo bằng `docker images | grep agent`:
> bản 1 stage base `python:3.11` đầy đủ nặng **1.69 GB**, bản multi-stage
> nặng **270 MB**, chênh nhau ~1.4 GB. Phần chênh đó là: (1) base image bản
> đầy đủ — Debian kèm compiler, header và dev tools; (2) toàn bộ build
> artifact của pip — cache, wheel trung gian, file tạm — chỉ cần để **cài**
> dependency chứ không cần để **chạy**. Dockerfile multi-stage của tôi dùng
> stage `builder` chỉ để `pip install --prefix=/install`, rồi stage runtime
> copy đúng phần đã cài sang base slim (`COPY --from=builder /install
> /usr/local`), nên image runtime không mang theo compiler và cache — nhỏ và
> an toàn hơn.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của tôi theo thứ tự: `FROM slim` → `COPY requirements.txt` →
> `RUN pip install` → `COPY app/` → `COPY utils/` → `USER` → `HEALTHCHECK` →
> `CMD`. Sửa một ký tự trong `app/main.py` rồi build lại:
>
> - **Dùng lại từ cache**: mọi layer trước `COPY app/` — base image,
>   `COPY requirements.txt`, `RUN pip install` — vì input của chúng không đổi.
> - **Phải chạy lại**: `COPY app/` (nội dung thay đổi) và mọi layer sau nó
>   (`COPY utils/`, `USER`, `HEALTHCHECK`, `CMD`).
>
> Nếu đặt `COPY . .` lên **trước** `RUN pip install`: chỉ sửa một ký tự ở bất
> kỳ file nào trong context (kể cả README) là `COPY . .` mất cache → `pip
> install` và toàn bộ layer sau phải chạy lại từ đầu → mỗi lần build chậm vài
> phút. Vì dependency (requirements.txt) hiếm khi đổi còn code thì đổi liên
> tục, để `pip install` trước `COPY` code giúp vòng lặp sửa code chỉ tốn vài
> giây.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện:
>
> 1. Code Python có lỗ hổng — ví dụ một endpoint giải tuần tự hóa input của
>    người dùng bằng `pickle.load`, hoặc lệnh `os.system` ghép chuỗi từ user
>    (command injection).
> 2. Kẻ tấn công exploit lỗ hổng để chạy code tùy ý **trong container**.
>    Container mặc định chạy bằng uid 0 (root) → code đó thực thi với quyền
>    root trong container.
> 3. Có quyền root trong container + một vector thoát (lỗi kernel, `docker.sock`
>    được mount vào, capability như `CAP_SYS_ADMIN`/`CAP_DAC_OVERRIDE`, thư
>    mục `/proc` hoặc `/sys` ghi được) → kẻ tấn công trèo ra khỏi namespace
>    container và thành root trên host.
> 4. Root trên host = đọc secrets, cài backdoor, chiếm toàn bộ máy.
>
> Lệnh `USER appuser` (uid 10001) trong Dockerfile của tôi cắt chuỗi ở bước
> **2 → 3**: dù kẻ tấn công có RCE, tiến trình chỉ chạy như một user thường
> với quyền hạn Linux tối thiểu — các kỹ thuật thoát container vốn cần root
> bên trong (mount `/proc/sys`, tạo namespace bằng `CAP_SYS_ADMIN`...) đều bị
> chặn hoặc bị hạn chế nặng. Hậu quả bị khoanh lại trong một app, không lan ra
> host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**.
>
> Cách đạt được: đếm theo phút đồng hồ thì bộ đếm reset vào giây :00. Tôi gửi
> 10 request lúc 10:00:59 (giây cuối của phút 10:00) và 10 request lúc
> 10:01:01 (giây đầu của phút 10:01). Xét theo từng phút, mỗi phút chỉ có 10
> request — "đúng luật" 10/phút. Nhưng tính theo thời gian thật, cả 20 request
> rơi vào khoảng 2 giây. Sliding window của tôi (`app/rate_limiter.py` dùng
> Redis ZSET, score = timestamp của request) đóng lỗ hổng này: tại 10:01:01 nó
> đếm mọi request có timestamp trong 60 giây trước, nên 10 request lúc
> 10:00:59 vẫn còn nằm trong cửa sổ → request thứ 11 bị chặn 429.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Khác nhau: rate limit (`app/rate_limiter.py`) giới hạn **số lượng request**
> trong cửa sổ 60 giây; cost guard (`app/cost_guard.py`) cộng dồn **số tiền
> thực tế** mỗi user tiêu trong tháng và chặn khi `spent + estimated_cost >
> monthly_budget_usd` (trả 402).
>
> - **Rate limit cho qua nhưng cost guard phải chặn**: user gọi chậm (dưới
>   10/phút) nhưng mỗi request đẩy prompt 50k token → chi phí mỗi call rất
>   cao. Đến giữa tháng tổng tiền vượt ngân sách $10 → cost guard chặn 402,
>   dù số request chưa bao giờ chạm hạn mức.
> - **Ngược lại**: user mới tiêu $0.5 (còn nhiều ngân sách) nhưng nện API 100
>   request trong 1 phút → rate limit chặn 429, dù tiền vẫn còn.
>
> Hai cơ chế bảo vệ hai tài nguyên khác nhau: rate limit chống spam/quá tải
> QPS, cost guard chống hóa đơn vượt trần.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Nếu gộp /health và /ready làm một và endpoint đó kiểm tra Redis, khi Redis
> mất kết nối 30 giây, thứ tự sự kiện:
>
> 1. Redis mất kết nối → cả 3 container trả 503 từ endpoint duy nhất (vì
>    `store.ping()` fail).
> 2. Platform thấy liveness probe của **cả 3** container fail → coi cả 3 chết →
>    kill và restart đồng loạt.
> 3. Trong lúc restart, không còn instance nào sống → mọi request 502/503 —
>    cụm sập hoàn toàn dù bản thân app Python vẫn khỏe mạnh.
> 4. Redis hồi phục sau 30s nhưng các container đang bị restart vòng vòng
>    (liveness lại fail nếu Redis vừa lên chưa kịp) → outage kéo dài hơn nhiều
>    so với 30 giây.
>
> Với thiết kế tách riêng như hiện tại: /health chỉ kiểm tra process sống
> (`app/main.py:76`) nên luôn 200 → container không bị giết; /ready trả 503 →
> load balancer ngừng đẩy traffic vào; khi Redis quay lại, /ready về 200 và
> traffic tự hồi. 30 giây Redis nghẽn chỉ là một nhịp, không thành outage.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, `history_length` tăng đều: 0, 1, 2, 3... — bất kể request rơi vào
> agent nào, vì lịch sử nằm trong Redis List dùng chung (`app/store.py`), cả 3
> instance đọc cùng một List.
>
> Nếu lịch sử nằm trong dict Python của từng process: mỗi instance giữ dict
> riêng. Load balancer chia request round-robin nên request sau thường rơi vào
> instance khác — instance đó không có tin nhắn trước → `history_length` sẽ
> nhảy lung tung, đa số là 0 hoặc 1, không bao giờ tăng đều. Container còn bị
> restart bất cứ lúc nào (deploy, OOM) thì dict mất sạch → agent "mất trí
> nhớ". Chính vì vậy state phải sống ngoài process — trong Redis — thì mới
> scale ngang được.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi tôi gặp: gọi `POST /ask` với API key hợp lệ trên bản deploy bị trả
> **500 Internal Server Error** — trong khi `/health` vẫn trả 200, nên service
> trông như đang chạy tốt.
>
> Nguyên nhân: biến `REDIS_URL` không được set trên cloud, nên app rơi vào giá
> trị mặc định `redis://localhost:6379/0` (`app/config.py:48`) — trỏ vào chính
> container app, nơi không chạy Redis. Auth pass (key đúng) → vào
> `limiter.check()` → gọi `zremrangebyscore` trên client Redis không kết nối
> được → `redis.exceptions.ConnectionError` không được bắt → FastAPI trả 500.
> `/health` không chạm Redis nên vẫn xanh — đúng cái bẫy của việc để health
> phụ thuộc dependency.
>
> Cách tìm: tôi mở log trên dashboard platform, thấy traceback
> `ConnectionError ... localhost:6379` ngay trong lần gọi /ask, và kiểm tra
> dashboard thấy đúng là chưa khai báo biến `REDIS_URL`.
>
> Sửa: tạo Redis add-on trên platform, copy connection string vào biến
> `REDIS_URL` trong dashboard (không commit vào repo), deploy lại → `/ask` trả
> 200 và `history_length` tăng dần sau mỗi câu hỏi.
