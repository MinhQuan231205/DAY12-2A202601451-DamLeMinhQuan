# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng `> *Câu trả lời của bạn*` bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: ..........................  Mã học viên: ..........................

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Tình huống: lúc deploy lên Render, tôi tạo Blueprint từ `render.yaml`
> nhưng nếu quên điền giá trị `AGENT_API_KEY` khi Render hỏi (biến này khai
> báo `sync: false` nên Render bắt phải nhập tay), container sẽ khởi động
> với biến rỗng. Vì `agent_api_key: str` không có default, Pydantic ném
> `ValidationError` ngay khi `Settings()` được gọi lúc import module — service
> crash tức thì, healthcheck báo fail, và tôi thấy ngay trong log Render là
> thiếu biến gì. Nếu để mặc định `"changeme"`, app vẫn khởi động và `/health`
> vẫn trả 200 bình thường — trông như mọi thứ ổn. Vấn đề chỉ lộ ra khi có ai
> đó (kể cả người lạ đọc được mã nguồn mở của repo) thử gọi `/ask` với key
> `"changeme"` và gọi được thật — lúc đó tôi mới phát hiện qua hóa đơn LLM
> tăng bất thường, tức là đã mất tiền trước khi biết có lỗi.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật lấy từ `uvicorn app.main:app` khi tôi gọi `/ask` với câu hỏi
> "Docker la gi?":
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:51:09.455490+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
>
> Hai việc làm được mà `print("đã trả lời xong")` không làm được:
>
> 1. **Lọc/tổng hợp theo trường** — vì mỗi dòng là JSON có key rõ ràng, tôi
>    có thể dùng `jq` hoặc query trên Datadog/Grafana kiểu
>    `SUM(cost_usd) WHERE event="ask_completed" GROUP BY user_id` để trả lời
>    "user nào tiêu nhiều tiền nhất hôm nay?" — với `print()` thì phải viết
>    regex đoán mò trên chuỗi tự do, dễ vỡ khi câu chữ đổi.
> 2. **Cảnh báo tự động (alerting)** — công cụ log gom log theo giờ có thể
>    đếm tỷ lệ `level: "error"` trên tổng số dòng trong 5 phút để tự bắn cảnh
>    báo Slack/PagerDuty khi vượt ngưỡng. `print()` không có `level` làm khóa
>    để đếm, nên không thể tự động hoá cảnh báo kiểu này.

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
| 1 stage (bản đầu) | 1730 MB (1.73 GB) |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Đo thật bằng `docker images day12-agent` trên máy tôi: bản 1-stage gốc
> (`FROM python:3.11` đầy đủ + `COPY . .` rồi `pip install` ngay trong đó)
> nặng **1.73GB**; bản multi-stage (builder cài dependency trên
> `python:3.11-slim`, runtime chỉ `COPY --from=builder /install /usr/local`
> + code, cũng trên `python:3.11-slim`) chỉ **270MB** — nhẹ hơn khoảng **6.4
> lần**, chênh lệch ~1.46GB.
>
> Phần dung lượng chênh lệch đó chủ yếu là:
> 1. **Base image đầy đủ vs slim**: `python:3.11` bao gồm rất nhiều gói hệ
>    thống (build tools, dev headers, docs, locale...) mà runtime không cần
>    dùng tới; `python:3.11-slim` bỏ hết những thứ đó, chỉ giữ Python
>    interpreter tối thiểu.
> 2. **Compiler/build tool không bị mang sang stage cuối**: các gói Python có
>    thành phần biên dịch (ví dụ phần C extension) cần `gcc`/`build-essential`
>    lúc `pip install`, nhưng multi-stage chỉ copy **kết quả cài đặt**
>    (`/install` → `/usr/local`) sang stage runtime, không copy compiler theo.
>    Bản 1-stage giữ nguyên toàn bộ toolchain build đó trong layer cuối cùng
>    dù không cần dùng lúc chạy.
> 3. **pip cache và file trung gian**: bản 1-stage không dùng `--no-cache-dir`
>    một cách triệt để tách biệt theo layer, nên các file tạm của quá trình
>    cài đặt vẫn tồn tại trong layer chung với source code.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của tôi (multi-stage): stage `builder` có `COPY requirements.txt .`
> rồi `RUN pip install ...`, xong mới `COPY app ./app` ở stage `runtime`. Khi
> tôi sửa một dòng trong `app/main.py` (thêm implementation cho `/ask`, `/ready`
> qua các lần build ở Block 3/4) và build lại, Docker log hiện `CACHED` cho toàn
> bộ layer của stage `builder` (`COPY requirements.txt .`, `RUN pip install
> --prefix=/install ...`) — layer này build lại từ đầu chỉ mất ~34s lần đầu,
> các lần build sau (không đổi `requirements.txt`) gần như tức thì vì dùng
> cache. Chỉ có layer `COPY app ./app` ở stage `runtime` (và các layer export
> ảnh phía sau nó) phải chạy lại, vì nội dung thư mục `app/` đã đổi.
>
> Nếu đặt `COPY . .` lên **trước** `RUN pip install` (giống bản gốc 1-stage
> lúc chưa sửa): mọi lần sửa dù chỉ 1 ký tự trong bất kỳ file nào cũng làm
> Docker coi layer `COPY . .` là "đã đổi" → **hủy cache từ đó trở đi**, kéo
> theo `RUN pip install -r requirements.txt` phải chạy lại toàn bộ (tải lại
> và cài lại ~30 package), dù `requirements.txt` không hề thay đổi. Với dự án
> này mất thêm khoảng 30 giây mỗi lần build — nhỏ ở đây nhưng với dependency
> nặng hơn (ví dụ PyTorch) có thể là vài phút mỗi lần chỉ vì sửa 1 dòng code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: (1) một lỗ hổng trong code Python — ví dụ một dependency có
> lỗi deserialization, hoặc một endpoint vô tình cho phép ghi file tùy ý — bị
> khai thác để kẻ tấn công thực thi lệnh shell bên trong container; (2) nếu
> container chạy bằng `root` (UID 0), lệnh thực thi đó cũng chạy với quyền
> root **bên trong** container; (3) container không phải một máy ảo hoàn
> toàn cô lập — nó dùng chung kernel với host qua namespace/cgroup. Nếu kẻ
> tấn công tìm được cách "thoát" container (kernel exploit, container
> misconfiguration, volume mount sai như mount `/` của host vào container,
> Docker socket bị lộ...), UID 0 bên trong container thường ánh xạ thẳng
> sang UID 0 (root thật) trên host — biến một lỗ hổng ứng dụng nhỏ thành
> quyền root toàn máy chủ.
>
> Lệnh `USER appuser` (UID 10001, không phải 0) trong `Dockerfile` cắt đứt
> chuỗi này ở bước (2): dù code Python bị chiếm quyền thực thi, tiến trình đó
> chỉ chạy với quyền của user thường bên trong container. Ngay cả khi kẻ tấn
> công thoát được ra host, UID 10001 đó không map sang root trên host — giới
> hạn thiệt hại lại ở mức "một user không có quyền gì đặc biệt", thay vì
> "root trên toàn máy chủ".

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa **20 request trong 2 giây**. Cách đạt được: gửi đúng 10 request lúc
> 10:00:59 (dùng hết quota của "phút 10:00"), rồi gửi tiếp 10 request lúc
> 10:01:01 (bộ đếm vừa reset về 0 cho "phút 10:01"). Cả 20 request đều "đúng
> luật" theo cách đếm cố định vì chúng thuộc hai cửa sổ đếm khác nhau, dù thực
> tế chỉ cách nhau 2 giây. Sliding window (ZSET) trong `app/rate_limiter.py`
> tránh được lỗ hổng này vì nó luôn nhìn lại đúng 60 giây gần nhất tính từ thời
> điểm request đến (`now - WINDOW_SECONDS`), không có mốc reset cố định nào để
> lợi dụng.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **số lượng** request theo thời gian (bao nhiêu lần/phút);
> cost guard giới hạn **số tiền** tích lũy theo tháng, bất kể tần suất gọi.
>
> - **Rate limit cho qua, cost guard phải chặn**: user chỉ gửi 2 request/phút
>   (dưới hạn mức 10/phút) nhưng mỗi câu hỏi rất dài (câu hỏi 2000 ký tự — sát
>   giới hạn `max_length` của `AskRequest`) khiến `tokens_in`/`tokens_out` lớn.
>   Sau vài chục request rải đều trong tháng, tổng `cost_usd` cộng dồn trong
>   `CostGuard.record()` vượt `MONTHLY_BUDGET_USD` dù tần suất gọi thấp — lúc
>   này `guard.check()` trả 402 dù `limiter.check()` vẫn cho qua bình thường.
> - **Cost guard cho qua, rate limit phải chặn**: user mới dùng, `spent()` còn
>   gần 0 (ngân sách dư dả), nhưng cùng lúc gửi 20 request liên tiếp trong vài
>   giây (ví dụ bot lỗi lặp vô hạn). `guard.check()` vẫn pass vì chưa tốn tiền
>   đáng kể, nhưng `limiter.check()` chặn 429 ngay từ request thứ 11 vì vượt
>   `RATE_LIMIT_PER_MINUTE` — bảo vệ hệ thống khỏi bị spam dù chưa tốn ngân
>   sách.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện:
>
> 1. Redis mất kết nối. Cả 3 container gọi `store.ping()` đều trả `False`
>    (vì `ping()` bọc try/except, không ném lỗi ra ngoài).
> 2. Endpoint gộp (giả sử vẫn tên `/health`) trả 503 ở **cả 3 container cùng
>    lúc**, vì cả 3 đều dùng chung một Redis.
> 3. Orchestrator (Docker/Railway/K8s) đọc `/health` là liveness probe — thấy
>    503 liên tục sau vài lần retry, nó kết luận "process chết" và **restart**
>    cả 3 container gần như đồng thời, dù process Python vẫn đang chạy bình
>    thường, chỉ là Redis chưa nối lại được.
> 4. Trong lúc cả 3 container đang restart (mất vài giây khởi động lại), cụm
>    có **0 instance nào phục vụ request** — downtime toàn phần dù nguyên nhân
>    gốc chỉ là Redis nấc 30 giây.
> 5. Khi Redis sống lại, container khởi động lại (đã restart) mới nối được,
>    nhưng thời gian downtime đã xảy ra oan uổng — một sự cố phụ thuộc nhỏ đã
>    bị khuếch đại thành sự cố toàn cụm.
>
> Tách riêng `/health` (không đụng Redis, chỉ dùng cho liveness) và `/ready`
> (có kiểm tra Redis, dùng cho readiness của load balancer) tránh được chuỗi
> sự kiện này: khi Redis chết, `/ready` báo 503 → load balancer **ngừng đẩy
> traffic mới vào**, nhưng `/health` vẫn 200 → orchestrator **không restart**
> container nào, chỉ chờ Redis hồi phục.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi build lại image (`docker compose up -d --build`) và gọi `/ask` 3 lần
> liên tiếp với cùng `X-User-Id: sv01` vào container thật (chưa kịp scale=3,
> nhưng nguyên lý giống hệt vì mỗi request là một HTTP call độc lập, không
> "dính" vào tiến trình nào). Kết quả `history_length` trả về đúng
> **0 → 2 → 4** — tăng đều mỗi lượt thêm 2 (user + assistant), khớp dữ liệu
> ghi trong Redis. Tôi cũng xác nhận bằng test
> `test_state_khong_nam_trong_process` (tests/test_cp4.py): dựng 2
> `ConversationStore` riêng biệt (mô phỏng 2 container khác tiến trình) trên
> cùng một Redis — instance B đọc thấy ngay dữ liệu instance A vừa ghi.
>
> Nếu lịch sử lưu trong dict Python thay vì Redis, mỗi container có RAM riêng
> nên `history_length` sẽ **không tăng đều đặn** — nó phụ thuộc container nào
> nhận request đó. Ví dụ: câu 1 vào container A → dict của A có 2 phần tử,
> `history_length=0` (lúc lấy history trước khi thêm câu này); câu 2 tình cờ
> vào container B (dict rỗng) → `history_length` lại là 0 thay vì 2 — agent
> "quên" mất câu 1. Số hiển thị sẽ nhảy lung tung (0, 0, 2, 0, 2, 4...) tùy
> load balancer route request vào container nào, thay vì tăng đơn điệu.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Ban đầu tôi chọn Railway (theo gợi ý của guide vì dễ nhất). Cài `@railway/cli`,
> `railway login`, `railway init`, `railway add --database redis`, set biến
> xong xuôi, nhưng khi chạy `railway up` thì lệnh fail ngay ở bước build với
> thông báo:
>
> ```
> Failed
> Your workspace has been restricted. Please attach a payment method or
> contact support to resolve this.
> ```
>
> Đây không phải lỗi trong code hay Dockerfile — build còn chưa kịp bắt đầu.
> Tìm nguyên nhân bằng cách đọc kỹ thông báo lỗi (rất rõ ràng, không cần đoán)
> và kiểm tra dashboard Railway: workspace mới tạo bị khóa vì chưa xác minh
> thẻ thanh toán, dù vẫn còn hạn mức $5 credit miễn phí — đây là chính sách
> chống lạm dụng của Railway, áp dụng trước cả khi build image.
>
> Cách sửa: thay vì gắn thẻ, tôi chuyển sang **Render** — platform thứ hai
> guide gợi ý, không đòi thẻ ngay cho free tier. Dùng `render.yaml` (Blueprint)
> có sẵn trong repo: vào render.com → New → Blueprint → chọn repo GitHub →
> Render tự đọc file, tạo cả service `day12-agent` và Redis `day12-redis` →
> điền `AGENT_API_KEY` khi được hỏi → Create. Build thành công ngay lần đầu,
> `/health` và `/ready` đều trả 200 sau khi deploy.
