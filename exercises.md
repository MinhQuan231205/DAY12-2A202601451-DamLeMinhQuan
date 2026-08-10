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

> Lúc deploy lên Render, em tạo Blueprint từ file render.yaml, biến
> AGENT_API_KEY được khai báo sync là false nên Render bắt em tự nhập tay
> giá trị đó. Nếu lúc đó em quên điền, container sẽ khởi động với biến
> trống. Vì agent_api_key không có giá trị mặc định, Pydantic báo lỗi
> ValidationError ngay khi Settings được gọi lúc import module, service sập
> tức thì và log trên Render chỉ thẳng ra biến nào đang thiếu, nên em sửa
> được ngay. Nếu để mặc định là changeme thì app vẫn khởi động bình thường
> và health vẫn trả về 200, nhìn bên ngoài mọi thứ có vẻ ổn. Vấn đề chỉ lộ
> ra khi có người đọc được mã nguồn công khai của repo, thử gọi ask với key
> changeme và gọi được thật, khi đó em mới phát hiện qua hóa đơn LLM tăng
> bất thường, nghĩa là đã mất tiền trước khi biết có lỗi xảy ra.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Log JSON thật em lấy được khi chạy uvicorn và gọi ask với câu hỏi Docker
> la gi như sau.
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T04:51:09.455490+00:00", "user_id": "sv01", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
>
> Có hai việc em làm được với dòng log này mà print đã trả lời xong không
> làm được. Việc thứ nhất là lọc và tổng hợp theo từng trường dữ liệu, vì
> mỗi dòng log là một JSON có khóa rõ ràng nên em có thể dùng công cụ như jq
> hoặc truy vấn trên Datadog để tính tổng cost_usd theo từng user_id, trả
> lời được câu hỏi user nào tiêu nhiều tiền nhất trong ngày, còn print chỉ
> ra một chuỗi tự do nên phải viết regex đoán mò và dễ vỡ khi câu chữ thay
> đổi. Việc thứ hai là cảnh báo tự động, công cụ gom log có thể đếm tỷ lệ
> level error trên tổng số dòng trong năm phút để tự bắn cảnh báo khi vượt
> ngưỡng, trong khi print không có trường level để làm khóa đếm nên không
> thể tự động hoá việc cảnh báo theo kiểu đó.

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
| 1 stage | 1730 MB |
| Multi-stage | 270 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> Em đo thật bằng lệnh docker images trên máy mình. Bản một stage dùng base
> image python:3.11 đầy đủ, copy toàn bộ code rồi mới chạy pip install,
> nặng 1.73 GB. Bản multi stage dùng builder cài dependency trên
> python:3.11 slim rồi runtime chỉ copy phần kết quả cài đặt sang, nặng
> 270 MB, nhẹ hơn khoảng 6.4 lần.
>
> Phần dung lượng chênh lệch đến từ ba chỗ chính. Một là base image đầy đủ
> mang theo rất nhiều gói hệ thống, công cụ build và tài liệu mà lúc chạy
> hoàn toàn không cần dùng, còn bản slim chỉ giữ lại phần Python interpreter
> tối thiểu. Hai là compiler và build tool không bị mang sang stage cuối, vì
> multi stage chỉ copy thư mục cài đặt sang runtime chứ không copy theo gcc
> hay build essential, còn bản một stage vẫn giữ nguyên toàn bộ công cụ
> build đó trong layer cuối dù không cần dùng lúc chạy. Ba là các file tạm
> và cache của quá trình cài đặt trong bản một stage vẫn nằm chung layer với
> source code, không tách biệt được như cách multi stage làm.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của em theo kiểu multi stage, stage builder copy requirements
> vào trước rồi mới chạy pip install, xong mới copy thư mục app sang stage
> runtime. Khi em sửa một dòng trong app/main.py rồi build lại, log Docker
> hiện cached cho toàn bộ layer của stage builder, gồm bước copy
> requirements.txt và bước pip install. Layer này chỉ mất khoảng ba mươi
> bốn giây ở lần build đầu tiên, những lần sau gần như tức thì vì dùng lại
> cache. Chỉ có layer copy thư mục app ở stage runtime và các layer export
> ảnh phía sau nó phải chạy lại, vì nội dung thư mục app đã thay đổi.
>
> Nếu đặt bước copy toàn bộ code lên trước bước pip install, giống bản gốc
> một stage lúc chưa sửa, thì mọi lần chỉnh dù chỉ một ký tự trong bất kỳ
> file nào cũng khiến Docker coi layer copy đó đã đổi và hủy cache từ đó
> trở đi, kéo theo bước pip install phải chạy lại toàn bộ, tải và cài lại
> khoảng ba mươi gói dù requirements.txt không hề thay đổi. Với dự án này
> thì mất thêm khoảng ba mươi giây mỗi lần build, còn với dependency nặng
> hơn như PyTorch có thể mất thêm vài phút chỉ vì sửa một dòng code.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện bắt đầu từ một lỗ hổng trong code Python, ví dụ một thư viện
> phụ thuộc có lỗi deserialization hoặc một endpoint vô tình cho phép ghi
> file tùy ý, bị khai thác để kẻ tấn công thực thi lệnh shell bên trong
> container. Nếu container chạy bằng root, lệnh thực thi đó cũng chạy với
> quyền root bên trong container. Container không phải một máy ảo cô lập
> hoàn toàn, nó dùng chung kernel với host thông qua namespace và cgroup,
> nên nếu kẻ tấn công tìm được cách thoát ra khỏi container bằng lỗ hổng
> kernel, cấu hình sai hoặc mount nhầm thư mục gốc của host vào container,
> UID 0 bên trong container thường ánh xạ thẳng sang UID 0 thật trên host,
> biến một lỗ hổng ứng dụng nhỏ thành quyền root toàn máy chủ.
>
> Lệnh USER appuser với UID 10001 trong Dockerfile của em cắt đứt chuỗi sự
> kiện đó ngay ở bước thứ hai. Dù code Python bị chiếm quyền thực thi, tiến
> trình đó chỉ chạy với quyền của một user thường bên trong container. Ngay
> cả khi kẻ tấn công thoát được ra host, UID 10001 không map sang root trên
> host, nên thiệt hại bị giới hạn lại ở mức một user không có quyền gì đặc
> biệt thay vì root trên toàn máy chủ.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa hai mươi request trong hai giây. Cách đạt được là gửi đúng mười
> request lúc 10:00:59 để dùng hết quota của phút đó, rồi gửi tiếp mười
> request lúc 10:01:01 khi bộ đếm vừa reset về không cho phút kế tiếp. Cả
> hai mươi request đều hợp lệ theo cách đếm cố định vì chúng thuộc hai cửa
> sổ đếm khác nhau, dù thực tế chỉ cách nhau hai giây. Sliding window dùng
> ZSET trong app/rate_limiter.py tránh được lỗ hổng này vì nó luôn nhìn lại
> đúng sáu mươi giây gần nhất tính từ thời điểm request đến, không có mốc
> reset cố định nào để lợi dụng.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn số lượng request theo thời gian, còn cost guard giới
> hạn số tiền tích lũy theo tháng bất kể tần suất gọi nhanh hay chậm.
>
> Một tình huống rate limit cho qua nhưng cost guard phải chặn là khi user
> chỉ gửi hai request mỗi phút, thấp hơn nhiều so với hạn mức mười request
> mỗi phút, nhưng mỗi câu hỏi lại rất dài, gần chạm giới hạn hai nghìn ký tự
> của AskRequest, khiến số token đầu vào và đầu ra đều lớn. Sau vài chục
> request rải đều trong tháng, tổng cost_usd cộng dồn trong
> CostGuard.record vượt MONTHLY_BUDGET_USD dù tần suất gọi vẫn thấp, lúc
> này guard.check trả về 402 dù limiter.check vẫn cho qua bình thường.
>
> Ngược lại, một tình huống cost guard cho qua nhưng rate limit phải chặn là
> khi user mới dùng, số tiền đã tiêu gần như bằng không nên ngân sách còn
> rất dư dả, nhưng lại gửi liên tiếp hai mươi request trong vài giây, ví dụ
> do một bot bị lỗi lặp vô hạn. guard.check vẫn cho qua vì chưa tốn tiền
> đáng kể, nhưng limiter.check chặn với mã 429 ngay từ request thứ mười một
> vì vượt RATE_LIMIT_PER_MINUTE, bảo vệ hệ thống khỏi bị spam dù ngân sách
> chưa hề bị ảnh hưởng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện diễn ra như sau. Đầu tiên Redis mất kết nối, cả ba
> container gọi store.ping đều trả về False vì hàm này đã bọc try except
> nên không ném lỗi ra ngoài. Endpoint gộp, giả sử vẫn giữ tên health, trả
> về 503 ở cả ba container cùng lúc vì cả ba đều dùng chung một Redis.
> Orchestrator đọc health như một liveness probe, thấy 503 liên tục sau vài
> lần thử lại thì kết luận process đã chết và tiến hành restart cả ba
> container gần như đồng thời, dù tiến trình Python vẫn đang chạy bình
> thường, chỉ là chưa nối lại được Redis. Trong lúc cả ba container đang
> khởi động lại, cụm không còn instance nào phục vụ được request, gây ra
> một khoảng downtime toàn phần dù nguyên nhân gốc chỉ là Redis nấc ba mươi
> giây. Khi Redis sống lại thì các container vừa restart mới nối được,
> nhưng thời gian downtime đã xảy ra một cách oan uổng, một sự cố phụ thuộc
> nhỏ đã bị khuếch đại thành sự cố toàn cụm.
>
> Việc tách riêng health chỉ dùng cho liveness và không đụng tới Redis, còn
> ready mới kiểm tra Redis và dùng cho readiness của load balancer, giúp
> tránh được chuỗi sự kiện này. Khi Redis chết thì ready báo 503 khiến load
> balancer ngừng đẩy traffic mới vào, còn health vẫn trả 200 nên
> orchestrator không restart container nào, chỉ cần chờ Redis hồi phục là
> xong.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Em build lại image bằng docker compose up detach build, rồi gọi ask ba
> lần liên tiếp với cùng X-User-Id là sv01 vào một container thật. Lúc đó
> em chưa kịp scale lên ba container, nhưng nguyên lý vẫn giống hệt vì mỗi
> request là một lệnh gọi HTTP độc lập, không gắn với bất kỳ tiến trình cụ
> thể nào. Kết quả history_length trả về đúng 0 rồi 2 rồi 4, tăng đều mỗi
> lượt thêm hai vì có cả câu hỏi của user và câu trả lời của assistant,
> khớp với dữ liệu ghi trong Redis. Em cũng xác nhận thêm bằng test
> test_state_khong_nam_trong_process trong tests/test_cp4.py, dựng hai
> ConversationStore riêng biệt để mô phỏng hai container khác tiến trình
> trên cùng một Redis, và instance thứ hai đọc thấy ngay dữ liệu mà instance
> thứ nhất vừa ghi.
>
> Nếu lịch sử được lưu trong một dict Python thay vì Redis thì mỗi container
> có vùng nhớ riêng, nên history_length sẽ không còn tăng đều đặn mà phụ
> thuộc vào việc request đó rơi vào container nào. Ví dụ câu hỏi đầu tiên
> vào container A khiến dict của A có hai phần tử, nhưng câu hỏi thứ hai lại
> tình cờ vào container B với dict rỗng nên history_length quay lại bằng 0
> thay vì 2, tức là agent quên mất câu hỏi đầu tiên. Con số hiển thị sẽ nhảy
> lộn xộn kiểu 0, 0, 2, 0, 2, 4 tùy vào việc load balancer định tuyến
> request vào container nào, thay vì tăng đơn điệu như khi dùng Redis.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Ban đầu em chọn Railway vì hướng dẫn của lab gợi ý đây là lựa chọn dễ
> nhất. Em cài railway cli, đăng nhập, tạo project, thêm Redis, set đầy đủ
> biến môi trường, nhưng khi chạy railway up thì lệnh fail ngay ở bước build
> với thông báo như sau.
>
> ```
> Failed
> Your workspace has been restricted. Please attach a payment method or
> contact support to resolve this.
> ```
>
> Đây không phải lỗi trong code hay Dockerfile vì bước build còn chưa kịp
> bắt đầu. Em tìm ra nguyên nhân bằng cách đọc kỹ thông báo lỗi, vốn đã rất
> rõ ràng, và kiểm tra thêm trên dashboard Railway thì thấy workspace mới
> tạo bị khóa do chưa xác minh thẻ thanh toán, dù vẫn còn nguyên năm đô la
> credit miễn phí. Đây là chính sách chống lạm dụng của Railway, áp dụng
> ngay từ trước khi image được build.
>
> Cách em sửa là chuyển sang Render, platform thứ hai mà lab cũng gợi ý và
> không đòi thẻ ngay cho gói miễn phí. Em dùng file render.yaml có sẵn trong
> repo, vào trang Render tạo Blueprint từ repo GitHub của mình, Render tự
> đọc file và tạo cả service day12-agent lẫn Redis day12-redis, em chỉ cần
> điền AGENT_API_KEY khi được hỏi rồi bấm tạo. Lần build đầu tiên đã thành
> công, health và ready đều trả về 200 sau khi deploy xong.
