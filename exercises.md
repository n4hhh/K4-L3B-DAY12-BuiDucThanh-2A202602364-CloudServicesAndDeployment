# Phiếu Phản Ánh — K4 Level 3B, Ngày 12

> **Bài làm cá nhân.** Các câu trả lời dưới đây dựa trên quá trình em trực tiếp
> triển khai và kiểm thử service trong lab.

Họ và tên: Bùi Đức Thành  
Mã học viên: 2A202602364

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

> Một tình huống cụ thể là lúc deploy service lên Railway nhưng quên cấu hình
> `AGENT_API_KEY`. Nếu chương trình dùng mặc định `"changeme"` thì container vẫn
> có thể khởi động và health check vẫn xanh, làm mình tưởng deployment đã đúng.
> Tuy nhiên endpoint `/ask` lúc đó lại được bảo vệ bằng một key mặc định rất dễ
> đoán và có thể bị người khác gọi. Với cách fail fast, lỗi cấu hình xuất hiện
> ngay khi Settings được tạo nên mình phát hiện vấn đề trước khi coi service là
> sẵn sàng. Theo mình cách này tốt hơn vì biến một lỗi bảo mật âm thầm thành một
> lỗi deployment nhìn thấy được ngay.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

> Một log của request `/ask` có dạng:
>
> `{"event":"ask_completed","level":"info","timestamp":"2026-09-29T04:39:00+00:00","user_id":"sv-test","tokens_in":3,"tokens_out":35,"cost_usd":0.00002145}`
>
> Thứ nhất, vì log có cấu trúc JSON nên hệ thống log có thể lọc hoặc group theo
> các field như `event`, `user_id` hay `level`, thay vì phải parse một chuỗi
> text tự do. Ví dụ mình có thể tìm toàn bộ `ask_completed` của một user.
>
> Thứ hai, log chứa các số liệu như `tokens_in`, `tokens_out` và `cost_usd`, nên
> có thể dùng để thống kê usage, theo dõi chi phí hoặc tạo dashboard/alert. Một
> câu `print("đã trả lời xong")` chỉ cho biết code đã chạy tới đó và gần như
> không cung cấp dữ liệu để phân tích tiếp.

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
| 1 stage (bản đầu) | 446 MB |
| Multi-stage | 63.9 MB |

>Bản single-stage có content size khoảng 446 MB, trong khi bản multi-stage chỉ
>khoảng 63.9 MB, giảm khoảng 382 MB. Phần dung lượng chênh lệch chủ yếu đến từ
>base image và những thành phần chỉ cần trong quá trình build nhưng không cần 
>ở runtime. Bản single-stage dùng python:3.11 đầy đủ và giữ toàn bộ môi trường
>sau khi cài dependency trong cùng một image.
>Ở bản multi-stage, dependency được cài trong stage builder, sau đó runtime chỉ
>copy phần dependency đã cài từ /install cùng source code cần thiết. Runtime 
>cũng dùng python:3.11-slim, nên không phải mang theo toàn bộ nội dung của Python
>image đầy đủ. Điều này vừa giảm kích thước image, vừa giảm attack surface của
>container production.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

> Dockerfile của mình copy `requirements.txt` và chạy `pip install` trước khi
> copy source code. Vì vậy khi chỉ sửa một ký tự trong `app/main.py`,
> `requirements.txt` không thay đổi nên layer copy requirements và layer cài
> dependency vẫn được lấy từ cache. Các layer trước phần copy source cũng được
> reuse. Layer `COPY app ./app` bị thay đổi và các bước sau nó phải được tạo
> lại.
>
> Nếu đặt `COPY . .` trước `RUN pip install`, chỉ cần thay đổi bất kỳ file nào
> trong source thì checksum của layer `COPY` thay đổi. Khi đó layer
> `RUN pip install` phía sau cũng mất cache và phải cài lại toàn bộ dependency,
> dù `requirements.txt` hoàn toàn không đổi. Điều này làm mỗi lần build sau
> khi sửa code chậm hơn rất nhiều.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

> Ví dụ endpoint Python có một lỗ hổng cho phép attacker thực thi command.
> Nếu process trong container đang chạy bằng root thì command của attacker
> cũng có quyền root bên trong container. Từ vị trí đó attacker có nhiều khả
> năng hơn để lợi dụng một cấu hình nguy hiểm, volume được mount sai hoặc một
> lỗ hổng của container runtime/kernel để tác động tới host.
>
> Trong Dockerfile của mình, mình tạo user `app` rồi dùng `USER app`. Vì vậy
> nếu application bị chiếm quyền điều khiển thì attacker ban đầu chỉ có quyền
> của user thường trong container chứ không có root. `USER` không bảo đảm rằng
> container không bao giờ bị escape, nhưng nó cắt bớt một bước leo thang quyền
> rất quan trọng và tuân theo nguyên tắc least privilege.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

> Người dùng có thể gửi tối đa 20 request trong khoảng 2 giây. Ví dụ họ gửi
> 10 request vào khoảng `10:00:59`, vẫn nằm trong quota của phút 10:00. Ngay
> sau khi đồng hồ sang `10:01:00`, counter reset và họ gửi tiếp 10 request
> trong quota của phút 10:01. Như vậy tổng cộng 20 request có thể lọt qua chỉ
> trong khoảng 2 giây.
>
> Sliding window 60 giây tránh được boundary problem này vì ở thời điểm
> 10:01:00 nó vẫn nhìn lại 60 giây gần nhất và vẫn thấy 10 request vừa gửi,
> thay vì reset counter chỉ vì số phút trên đồng hồ thay đổi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

> Rate limit giới hạn tốc độ/số lượng request trong một khoảng thời gian ngắn,
> còn cost guard giới hạn tổng tiền đã sử dụng trong khoảng thời gian dài hơn,
> ở bài này là theo tháng.
>
> Trường hợp rate limit cho qua nhưng cost guard chặn là user chỉ gửi một vài
> request trong phút nên chưa vượt 10 request/phút, nhưng trước đó đã dùng gần
> hết ngân sách tháng. Request tiếp theo có thể vẫn đúng rate limit nhưng bị
> cost guard từ chối vì vượt budget.
>
> Trường hợp ngược lại là user mới bắt đầu tháng và gần như chưa tốn ngân sách,
> nhưng gửi hơn 10 request liên tiếp trong vòng 60 giây. Tổng cost vẫn rất thấp
> nên cost guard có thể cho qua, nhưng rate limiter phải trả `429 Too Many
> Requests`.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

> Đầu tiên Redis mất kết nối. Nếu `/health` cũng phụ thuộc vào Redis thì health
> check của cả ba container bắt đầu fail cùng lúc dù bản thân process FastAPI
> vẫn đang chạy bình thường.
>
> Tiếp theo orchestrator coi cả ba instance là unhealthy và có thể restart
> chúng. Nhưng nguyên nhân thực tế nằm ở Redis dùng chung nên restart container
> không sửa được gì. Các container mới khởi động lại vẫn không kết nối được
> Redis và tiếp tục fail health check, có thể tạo thành một vòng restart.
>
> Trong khoảng đó service có nguy cơ mất toàn bộ instance đang phục vụ, dù lỗi
> chỉ nằm ở dependency bên ngoài. Khi Redis hoạt động lại, các container mới
> bắt đầu trở về trạng thái healthy.
>
> Tách `/health` và `/ready` tránh tình trạng này. Khi Redis chết, `/ready` có
> thể trả 503 để instance tạm thời không nhận traffic, còn `/health` vẫn cho
> biết process còn sống nên orchestrator không cần restart hàng loạt.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

> Với Redis, cả ba instance dùng chung một nơi lưu state nên request có rơi vào
> container nào thì cũng đọc được cùng một history. Trong implementation của
> mình, mỗi request thành công append hai message là `user` và `assistant`, còn
> `history_length` được lấy trước khi append. Vì vậy nó tăng theo dạng
> `0, 2, 4, 6, ...` cho cùng một user, cho tới giới hạn history.
>
> Nếu thay Redis bằng một dict Python thì mỗi container có một dict riêng.
> Khi load balancer chuyển các request qua ba instance, container B sẽ không
> thấy history vừa được ghi ở container A. Với round-robin đơn giản có thể thấy
> dạng như `0, 0, 0, 2, 2, 2, ...` thay vì một history chung tăng liên tục.
> Nếu routing không đều thì giá trị còn có thể nhìn như bị tăng rồi giảm ngẫu
> nhiên. Khi một container restart, toàn bộ history trong dict của container đó
> cũng mất. Đây là lý do state phải nằm ngoài process nếu muốn scale ngang.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

> Khi deploy lên Railway, sau khi `railway up --service day12-agent` upload xong
> thì CLI báo:
>
> `reqwest error: error sending request for url (https://backboard.railway.com/graphql/v2) - operation timed out`
>
> Ban đầu mình tưởng deployment bị fail, nhưng khi chạy
> `railway deployment list --service day12-agent --limit 5` thì deployment lại
> có trạng thái `SUCCESS`. Sau đó `railway logs` tiếp tục lỗi connect/DNS.
>
> Khi đã generate public domain, `curl` còn báo
> `Could not resolve host: day12-agent-production-5931.up.railway.app`.
> Mình kiểm tra bằng `nslookup` và thấy DNS mặc định đang là DNS nội bộ của
> VinUni (`VU-AD02-VP.vinuni.local`) và request bị timeout. Query qua
> `1.1.1.1` cuối cùng vẫn resolve được domain Railway, nên mình xác định đây
> không phải lỗi application hay Redis mà là vấn đề DNS/network ở máy đang dùng.
>
> Mình chuyển sang một mạng khác rồi gọi lại public URL. Sau đó `/health` trả
> `HTTP 200` với `{"status":"ok","service":"day12-agent","version":"1.0.0"}`
> và `/ready` cũng trả `HTTP 200` với `{"status":"ready","redis":true}`. Qua lỗi
> này mình thấy trạng thái deployment, health của application và lỗi kết nối
> từ client là ba lớp khác nhau; không nên thấy CLI timeout rồi kết luận ngay
> rằng service trên cloud đã deploy thất bại.