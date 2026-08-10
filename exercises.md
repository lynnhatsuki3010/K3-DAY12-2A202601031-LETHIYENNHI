# Phiếu Phản Ánh — K3 Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng trích dẫn mẫu (bắt đầu bằng `>`) bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Lê Thị Yến Nhi  Mã học viên: 2A202601031

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Lúc deploy lên Railway, quên set `AGENT_API_KEY` trước khi chạy `railway up`
> lần đầu. Vì `agent_api_key: str` không có giá trị mặc định, app báo lỗi
> `ValidationError` ngay khi khởi động và container không lên được, thấy
> ngay trong log Railway. Nếu field này có mặc định kiểu `"changeme"`, app vẫn
> chạy bình thường, `/health` vẫn trả 200, tưởng đã deploy thành công. Nhưng
> lúc đó bất kỳ ai gửi header `X-API-Key: changeme` cũng gọi được `/ask` miễn
> phí bằng "khóa" của tôi và chỉ phát hiện ra khi xem hóa đơn hoặc log bất
> thường tức là sau khi thiệt hại đã xảy ra, không phải lúc còn đang ngồi
> nhìn màn hình deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Dòng log thật tôi thu được khi gọi `/ask`:
>
> ```json
> {"event": "ask_completed", "level": "info", "timestamp": "2026-08-10T03:23:50+00:00", "user_id": "sv-test", "tokens_in": 3, "tokens_out": 41, "cost_usd": 2.505e-05}
> ```
>
> Hai việc `print("đã trả lời xong")` không làm được: (1) lọc/tổng hợp theo
> trường — ví dụ `grep user_id=sv01 | jq .cost_usd` để tính user nào tiêu nhiều
> tiền nhất trong ngày, print không có cấu trúc nên không tách được field; (2)
> đếm tỷ lệ lỗi tự động — công cụ log (Datadog, Railway logs) có thể đếm số
> dòng `"level":"error"` trong 5 phút gần nhất và tự bắn cảnh báo, còn chuỗi
> text tự do thì máy không biết đâu là "lỗi" đâu là "bình thường".

---

### Câu 3 — Kích thước image (CP2)

Build cả hai phiên bản và ghi lại số đo thật:

```bash
docker build -f <Dockerfile-1-stage> -t agent:single .
docker build -t agent:multi .
docker images | grep agent
```

| Bản                 | Dung lượng |
| -------------------- | ------------ |
| 1 stage (bản đầu) | ... MB       |
| Multi-stage          | ... MB       |

Giải thích: phần dung lượng chênh lệch đó là những gì?

> | Bản                                                         | Dung lượng |
> | ------------------------------------------------------------ | ------------ |
> | 1 stage (`python:3.11`, `COPY . .` rồi `pip install`) | 1.73 GB      |
> | Multi-stage (`python:3.11-slim`, tách builder/runtime)    | 270 MB       |
>
> Chênh lệch ~1.46GB đến từ ba chỗ: (1) base image — `python:3.11` đầy đủ mang
> theo cả bộ công cụ biên dịch (gcc, make...) và rất nhiều package hệ điều hành
> Debian không dùng tới, còn `slim` cắt gần hết; (2) toàn bộ cache/artifact của
> `pip install` (mã nguồn `.tar.gz` tải về, file build trung gian khi biên dịch
> các gói có phần C) bị giữ lại trong layer của bản 1-stage, còn bản multi-stage
> stage `builder` làm xong bị vứt, chỉ `COPY --from=builder /install` phần kết
> quả cài đặt sang; (3) `--no-cache-dir` khi pip install ở bản multi-stage
> không lưu cache trong image.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Sửa một dòng trong `app/main.py` rồi `docker build` lại: layer
> `COPY requirements.txt .` và `RUN pip install --prefix=/install ...` ở stage
> `builder` được lấy từ cache (Docker thấy `requirements.txt` không đổi nội
> dung, hash layer giống hệt cũ), chỉ có layer `COPY app ./app` trở về sau (và
> `RUN chown`) phải chạy lại — build xong trong vài giây. Nếu đặt `COPY . .`
> lên trước `RUN pip install`: mỗi lần sửa dù chỉ một ký tự trong code, hash
> của layer `COPY . .` đổi, Docker coi mọi layer sau nó là "bẩn" và phải chạy
> lại từ đó — kể cả `pip install`, nghĩa là cài lại toàn bộ thư viện (tốn hàng
> chục giây tới vài phút) chỉ vì một dòng code không liên quan gì tới
> dependency.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Chuỗi sự kiện: một lỗ hổng trong code Python (ví dụ thư viện deserialize dữ
> liệu không tin cậy, hoặc dependency có lỗi RCE) bị khai thác → kẻ tấn công
> chạy được lệnh hệ điều hành bên trong container → nếu process đang chạy bằng
> `root`, lệnh đó có toàn quyền trong container: ghi đè file hệ thống, cài
> backdoor, đọc mọi secret nằm trong biến môi trường → nếu container có lỗ hổng
> escape (kernel bug, volume mount cấu hình sai, socket Docker bị lộ), quyền
> root trong container leo thẳng thành quyền root trên máy host thật.
> Lệnh `USER appuser` (uid 10001) cắt chuỗi ngay ở bước thứ hai: dù exploit vẫn
> chạy lệnh được, nó chỉ có quyền của một user thường — không ghi được vào thư
> mục hệ thống, không phải root nên dù có escape container cũng không tự động
> thành root trên host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Tối đa 20 request trong 2 giây. Cách đạt được: gửi đúng 10 request vào lúc
> phút X giây 59 (bộ đếm của phút X đang ở 0/10, 10 request này lấp đầy hạn
> mức của phút X), rồi gửi tiếp 10 request nữa vào giây 00–01 của phút X+1 (bộ
> đếm vừa reset về 0 nên lại được phép đủ 10 request). Với cách đếm theo phút
> đồng hồ, cả hai đợt đều "hợp lệ" riêng lẻ, nhưng gộp lại là 20 request chỉ
> trong khoảng 1–2 giây thực tế — đúng lỗ hổng mà sliding window (đếm 60 giây
> gần nhất tính từ thời điểm request, không phải theo mốc phút cố định) không
> mắc phải.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn **tần suất** (bao nhiêu request trong một khoảng thời
> gian), cost guard giới hạn **số tiền** (tổng chi phí cộng dồn trong tháng),
> hai đại lượng độc lập nhau. Tình huống rate limit cho qua nhưng cost guard
> chặn: user chỉ gửi 3 request/phút (thấp hơn hạn mức 10/phút nhiều), nhưng mỗi
> câu hỏi rất dài (hàng chục nghìn token) khiến chỉ vài chục request trong
> tháng đã vượt `MONTHLY_BUDGET_USD` — rate limit không có lý do gì để chặn,
> cost guard mới là lớp chặn được. Tình huống ngược lại: user gửi đúng 15
> request/phút với câu hỏi rất ngắn (gần như không tốn tiền) — rate limit chặn
> ở request thứ 11 (429) dù tổng chi phí cả tháng còn cách xa ngân sách, cost
> guard nếu đứng một mình sẽ vẫn cho qua.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Thứ tự sự kiện nếu gộp `/health` = `/ready` (cả hai đều check Redis):
> (1) Redis mất kết nối → cả 3 container cùng lúc gọi `store.ping()` trong
> endpoint health thất bại → cả 3 cùng trả 503; (2) orchestrator coi 503 ở
> endpoint liveness là "process chết", ra lệnh **restart** cả 3 container gần
> như đồng thời (đây là điểm khác biệt chính so với thiết kế đúng: đáng lẽ chỉ
> `/ready` báo lỗi để load balancer rút traffic, container vẫn sống); (3) trong
> lúc cả 3 đang restart, không còn container nào chạy để phục vụ request —
> dịch vụ sập hoàn toàn dù chưa từng có container nào thực sự bị treo hay
> crash; (4) 30 giây sau Redis hồi phục, nhưng 3 container vẫn đang trong chu
> trình khởi động lại (kéo image, chạy lifespan, chờ healthcheck) nên downtime
> kéo dài hơn cả thời gian Redis mất kết nối thực tế. Một sự cố nhỏ ở Redis
> biến thành sự cố toàn hệ thống.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Tôi đã chạy thật `docker compose up -d --scale agent=3` (vì compose map cứng
> `8000:8000` nên phải đổi sang dải cổng `8000-8010:8000` để 3 container không
> tranh cổng), rồi gọi `/ask` 5 lần với cùng `X-User-Id` nhưng đổi cổng
> (container) mỗi lần: `history_length` trả về lần lượt `0 → 2 → 4 → 6 → 8`,
> tăng đều dù request rơi vào container 1, 2 hay 3 — vì lịch sử nằm trong Redis
> dùng chung. Nếu lưu trong một dict Python (RAM riêng của từng process) thay
> vì Redis: mỗi container có bộ nhớ tách biệt, `history_length` sẽ không tăng
> đều nữa mà nhảy lộn xộn theo container nào nhận request — ví dụ request 1 và
> 4 cùng rơi vào container A thì A báo `history_length=2`, còn request 2, 3, 5
> rơi vào B/C sẽ báo `history_length=0` hoặc `2` dù thực tế đã hỏi 5 lần —
> agent trông như "mất trí nhớ" ngẫu nhiên.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Lỗi gặp khi deploy lên Railway: container liên tục crash-restart, healthcheck
> `/health` timeout, `Deploy failed`. Thông báo lỗi trong log:
> `Error: Invalid value for '--port': '$PORT' is not a valid integer.`
> Nguyên nhân: tôi đặt `startCommand = "uvicorn app.main:app --host 0.0.0.0 --port $PORT"` trong `railway.toml`. Railway chạy `startCommand` trực tiếp
> (không qua shell), nên `$PORT` không được hệ điều hành nội suy thành số cổng
> thật — uvicorn nhận đúng chuỗi ký tự `"$PORT"` làm tham số `--port` và báo
> lỗi vì đó không phải số nguyên. Tôi tìm ra bằng cách chạy `railway logs` để
> đọc traceback thực tế thay vì đoán. Cách sửa: xóa hẳn dòng `startCommand`
> trong `railway.toml`, để Railway dùng thẳng `CMD` đã viết đúng trong
> `Dockerfile` (`sh -c "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"`) — vì có `sh -c` nên biến môi trường được shell nội suy đúng
> trước khi truyền cho uvicorn.
