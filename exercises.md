# Phiếu Phản Ánh — K4 Level 3A, Ngày 12

> **Bài làm cá nhân.** Trả lời bằng lời của chính bạn, dựa trên những gì bạn
> quan sát được khi chạy code — không sao chép đáp án của người khác.
>
> Cách trả lời: thay dòng placeholder tương ứng bằng câu trả lời.
> `grade.py` đếm số câu đã trả lời (15 điểm cho 10 câu).
>
> Họ và tên: Đỗ Khắc Gia Khoa  Mã học viên: 2A202602733

---

### Câu 1 — Fail fast (CP1)

Trong `Settings`, `agent_api_key` không có giá trị mặc định nên app chết ngay
khi khởi động nếu thiếu biến môi trường. Hãy mô tả một tình huống cụ thể mà
việc "chết sớm" này cứu bạn, so với việc để mặc định `"changeme"`.

**Tình huống thực tế:** Khi deploy bản cập nhật lên môi trường staging/production (như Render, Railway, Cloud Run), kỹ sư cấu hình quên khai báo biến môi trường `AGENT_API_KEY` trong dashboard hoặc secret manager.
- **Nếu để mặc định `"changeme"`**: Ứng dụng vẫn khởi động thành công và báo healthy. Bất kỳ ai hoặc bot quét Internet gọi API với key `"changeme"` đều được sử dụng miễn phí, gây rò rỉ tài nguyên LLM. Đồng thời, toàn bộ client hợp lệ gửi API key thật đều bị lỗi 401 Unauthorized, làm gián đoạn dịch vụ mà hệ thống giám sát vẫn báo container bình thường. Hậu quả là chỉ phát hiện sự cố khi khách hàng khiếu nại hoặc nhận hóa đơn chi phí LLM khổng lồ.
- **Khi áp dụng Fail-Fast (không có mặc định)**: Ứng dụng ném `ValidationError` và crash ngay lúc khởi động tiến trình. Hệ thống điều phối (orchestrator) bắt được exit code lỗi ngay lập tức, kích hoạt cảnh báo deployment failure và rollback về bản cũ, bảo vệ ngân sách và buộc kỹ sư phải cấu hình secret ngay khi deploy.

---

### Câu 2 — Log cho máy đọc (CP1)

Chạy service và gọi `/ask` vài lần. Dán một dòng log JSON bạn thu được, rồi
nêu **hai** việc bạn làm được với dòng log đó mà `print("đã trả lời xong")`
không làm được.

**Dòng log JSON thực tế:**
```json
{"event": "ask_completed", "level": "info", "timestamp": "2026-09-28T08:15:30.123456+00:00", "user_id": "sv-test", "tokens_in": 12, "tokens_out": 45, "cost_usd": 0.000114}
```

**Hai việc làm được với log JSON mà `print()` không làm được:**
1. **Truy vấn, lọc và thống kê định lượng theo trường dữ liệu (Structured Log Aggregation & Metrics)**: Các hệ thống thu thập log tập trung (như Datadog, Grafana Loki, CloudWatch, ELK) có thể tự động parse JSON để thực hiện các phép tính phân tích: tính tổng chi phí (`cost_usd`) theo từng `user_id` trong ngày, đo lường lượng token trung bình tiêu thụ trên mỗi request, hoặc lọc tìm top người dùng tiêu tốn nhiều tài nguyên nhất.
2. **Thiết lập hệ thống cảnh báo tự động theo ngưỡng (Automated Alerting & Threshold Monitoring)**: Có thể định nghĩa các quy tắc giám sát tự động kích hoạt alert khi xuất hiện bất thường (ví dụ: cảnh báo khi `cost_usd > 0.05` trên 1 request duy nhất để phát hiện prompt injection/lạm dụng API, hoặc theo dõi tỷ lệ `level == "error"` vượt quá 5% trong 5 phút để thông báo ngay về kênh Slack của đội ngũ kỹ thuật).

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
| 1 stage (bản đầu) | 1.02 GB |
| Multi-stage | 185 MB |

Giải thích: phần dung lượng chênh lệch đó là những gì?

**Phần dung lượng chênh lệch (~835 MB) bao gồm:**
1. **Hệ điều hành và công cụ phát triển không cần thiết ở runtime**: Base image đầy đủ `python:3.11` chứa Debian chuẩn cùng các công cụ build (`gcc`, `g++`, `make`), headers (`build-essential`, `libssl-dev`), tài liệu man pages, và các tiện ích quản trị. Trong khi đó, `python:3.11-slim` chỉ giữ lại kernel user-space và các thư viện runtime tối thiểu để chạy Python interpreter.
2. **Pip cache và build artifacts**: Ở mô hình single-stage, toàn bộ cache tải wheel của `pip`, file trung gian khi compile thư viện đều bị lưu vĩnh viễn vào các layer của image. Trong khi ở multi-stage, stage `builder` đảm nhận toàn bộ quá trình tải và compile, sau đó chỉ copy thư mục kết quả `/install` sang `/usr/local` ở stage `runtime`, loại bỏ hoàn toàn compiler và cache tải về khỏi image cuối cùng.

---

### Câu 4 — Thứ tự lệnh trong Dockerfile (CP2)

Sửa một ký tự trong `app/main.py` rồi build lại. Với Dockerfile của bạn, những
layer nào được dùng lại từ cache, layer nào phải chạy lại? Nếu bạn đặt
`COPY . .` lên trước `RUN pip install` thì kết quả khác thế nào?

- **Với Dockerfile hiện tại**:
  - Các layer từ đầu đến trước lệnh `COPY app ./app` (bao gồm: nạp base image, tạo workdir, `COPY requirements.txt .`, `RUN pip install ...`, copy `/install` sang runtime, tạo `appuser`) đều được dùng lại hoàn toàn từ Docker layer cache (`CACHED`).
  - Chỉ các layer từ `COPY app ./app` trở đi mới phải thực thi lại. Do đó quá trình build lại diễn ra gần như tức thì (~1-2 giây).
- **Nếu đặt `COPY . .` lên trước `RUN pip install`**:
  - Mỗi khi sửa dù chỉ 1 ký tự trong `app/main.py`, checksum của layer `COPY . .` thay đổi làm toàn bộ cache từ điểm đó trở đi bị vô hiệu hóa (cache invalidated).
  - Lệnh `RUN pip install` bắt buộc phải chạy lại từ đầu (tải lại toàn bộ dependencies từ Internet). Điều này khiến mỗi lần sửa code nhỏ đều tốn vài phút build lại, tiêu tốn băng thông và làm chậm đáng kể chu kỳ release.

---

### Câu 5 — Vì sao không chạy bằng root (CP2)

Container mặc định chạy bằng root. Mô tả chuỗi sự kiện dẫn từ "một lỗ hổng
trong code Python của bạn" tới "kẻ tấn công có quyền cao trên máy host", và
lệnh `USER` cắt đứt chuỗi đó ở chỗ nào.

**Chuỗi sự kiện leo thang đặc quyền (Privilege Escalation):**
1. Code Python gặp lỗi bảo mật (như Remote Code Execution qua `eval()`, deserialization không an toàn qua `pickle`, hoặc command injection qua lệnh shell).
2. Kẻ tấn công kích hoạt mã độc từ xa, chiếm quyền thực thi lệnh bên trong container.
3. Vì container mặc định chạy bằng `root` (UID 0), kẻ tấn công sở hữu quyền root trong namespace của container. Nếu container có mount Docker socket (`/var/run/docker.sock`), mount thư mục nhạy cảm từ host, hoặc kernel Linux có lỗ hổng container breakout (như CVE-2019-5736, Dirty COW), tiến trình root này có thể thao túng daemon hoặc thoát khỏi cgroup/namespace để chiếm quyền điều khiển root trực tiếp trên máy host vật lý.
- **Vị trí lệnh `USER appuser` cắt đứt chuỗi:** Lệnh `USER appuser` chuyển tiến trình sang user không đặc quyền (UID 10001). Khi kẻ tấn công khai thác được RCE trong code Python, shell thu được chỉ có đặc quyền hạn chế của `appuser` — không thể ghi đè file hệ thống, không có quyền thao tác với docker socket hoặc gọi các system calls nguy hiểm, ngăn chặn hoàn toàn nguy cơ container breakout lên máy host.

---

### Câu 6 — Cửa sổ trượt (CP3)

Rate limit của bạn dùng sliding window 60 giây. Nếu thay bằng cách đếm theo
phút đồng hồ (reset lúc giây 00), một người dùng có thể gửi tối đa bao nhiêu
request trong 2 giây liên tiếp khi hạn mức là 10/phút? Giải thích cách đạt được
con số đó.

- **Số request tối đa:** **20 request** trong 2 giây liên tiếp.
- **Giải thích cách đạt được:**
  - Cơ chế đếm theo phút đồng hồ (Fixed Window Counter) reset quota về 0 ở mỗi mốc tròn phút (giây `:00`).
  - Người dùng gửi dồn dập 10 request ở giây cuối cùng của phút thứ nhất: lúc `10:00:59` (hợp lệ vì thuộc hạn mức của phút 10:00).
  - Ngay 1 giây sau đó, thời gian chuyển sang `10:01:00`, bộ đếm được reset về 0 cho phút mới.
  - Người dùng lập tức gửi tiếp 10 request nữa ở giây `10:01:00` (vẫn hợp lệ theo hạn mức của phút 10:01).
  - Kết quả: Trong khoảng thời gian từ `10:00:59` đến `10:01:00` (đúng 2 giây liên tiếp), người dùng đã gửi thành công tổng cộng 20 request, gây ra một đợt bùng nổ lưu lượng (traffic spike) gấp đôi hạn mức hệ thống cho phép. Sliding Window loại bỏ hoàn toàn kẽ hở này nhờ tính toán chính xác số request trong đúng 60 giây trượt lùi từ thời điểm gửi.

---

### Câu 7 — Rate limit và cost guard (CP3)

Hai cơ chế này khác nhau ở điểm nào? Cho một tình huống mà rate limit cho qua
nhưng cost guard phải chặn, và một tình huống ngược lại.

**Sự khác nhau cơ bản:**
- **Rate Limit**: Giới hạn **tần suất/tốc độ** request trong thời gian ngắn (ví dụ: số request/phút) để chống quá tải tài nguyên máy chủ, nghẽn mạng và tấn công từ chối dịch vụ (DDoS).
- **Cost Guard**: Giới hạn **chi phí tài chính thực tế** trong chu kỳ dài (ví dụ: số USD/tháng) dựa trên lượng tiêu thụ tài nguyên thực tế (số lượng token input/output của mô hình LLM).

**Hai tình huống minh họa:**
1. **Rate limit cho qua nhưng Cost guard chặn:** Người dùng chỉ gửi 1 request mỗi 10 phút (hoàn toàn dưới ngưỡng 10 request/phút của Rate Limiter). Tuy nhiên, mỗi request lại tải lên một prompt dài 100.000 tokens và yêu cầu LLM phân tích chuyên sâu với chi phí $0.5/request. Sau 20 request trong tháng (đạt mốc $10.0), Cost Guard sẽ trả về lỗi 402 Payment Required để bảo vệ ngân sách, dù tần suất gọi API của người dùng rất thấp.
2. **Cost guard cho qua nhưng Rate limit chặn:** Người dùng mới bắt đầu tháng với ngân sách còn nguyên $10.0. Người này chạy script spam liên tiếp 15 request "Xin chào" cực ngắn (mỗi request chỉ 10 tokens, chi phí chỉ $0.00001, tổng 15 request chưa đến $0.0002 — hoàn toàn không đe dọa ngân sách tháng). Tuy nhiên, do gửi 15 request trong vòng vài giây, vi phạm ngưỡng 10 request/phút, Rate Limiter sẽ lập tức chặn lại với mã lỗi 429 Too Many Requests để bảo vệ hạ tầng.

---

### Câu 8 — /health khác /ready (CP4)

Nếu gộp hai endpoint làm một và cho nó kiểm tra Redis, chuyện gì xảy ra với cụm
3 container khi Redis mất kết nối 30 giây? Trả lời theo đúng thứ tự sự kiện.

**Thứ tự sự kiện xảy ra sự cố dây chuyền (Cascading Failure):**
1. **Giây 00**: Redis gặp sự cố gián đoạn mạng tạm thời hoặc khởi động lại trong 30 giây.
2. **Giây 05**: Orchestrator (Docker Swarm/Kubernetes/Cloud Agent) gửi liveness probe định kỳ tới cả 3 container agent. Do liveness probe kiểm tra Redis và Redis đang mất kết nối, endpoint trả về lỗi (503 hoặc timeout) trên toàn bộ 3 container.
3. **Giây 10 - 15**: Vì liveness probe báo fail, orchestrator hiểu nhầm rằng tiến trình ứng dụng đã chết hoàn toàn và ra lệnh restart đồng loạt cả 3 container.
4. **Giây 15 - 30**: Khi container khởi động lại, chúng tiếp tục thực hiện probe kiểm tra Redis ngay lúc khởi động; do Redis vẫn chưa sẵn sàng, các container tiếp tục fail và rơi vào vòng lặp khởi động lại liên tục (CrashLoopBackOff).
5. **Giây 30**: Redis đã phục hồi và hoạt động bình thường trở lại. Tuy nhiên, toàn bộ 3 container agent đều đang kẹt trong chu kỳ restart của orchestrator, khiến toàn bộ hệ thống bị sập hoàn toàn (downtime 100%), người dùng nhận lỗi 502 Bad Gateway.
*Bài học:* `/health` (liveness) chỉ kiểm tra process sống chết, còn `/ready` (readiness) mới kiểm tra dependency để điều phối traffic của Load Balancer.

---

### Câu 9 — Stateless (CP4)

Chạy `docker compose up --scale agent=3` rồi gọi `/ask` nhiều lần với cùng một
`X-User-Id`. Quan sát `history_length` trong response. Nếu lịch sử được lưu
trong một dict Python thay vì Redis, bạn sẽ thấy con số đó thay đổi thế nào?

- **Khi lưu trên Redis (Stateless):**
  Lịch sử hội thoại được tập trung tại Redis. Dù Load Balancer điều phối các request lần lượt qua các container A, B, C khác nhau, mọi instance đều đọc/ghi chung một nguồn dữ liệu. Vì vậy `history_length` tăng đều đặn và nhất quán: 0, 2, 4, 6, 8... (mỗi lượt tăng 1 message của user và 1 message của assistant).
- **Nếu lưu trong dict Python (Stateful trong RAM tiến trình):**
  Mỗi container có một vùng nhớ RAM tách biệt hoàn toàn. Khi request được phân phối ngẫu nhiên (round-robin) qua 3 container:
  - Lượt 1 vào container A: lưu vào dict của A -> `history_length = 0`.
  - Lượt 2 vào container B: dict của B rỗng -> `history_length = 0` (thay vì 2).
  - Lượt 3 vào container C: dict của C rỗng -> `history_length = 0` (thay vì 4).
  - Lượt 4 vào lại container A: dict của A có 2 tin nhắn -> `history_length = 2`.
  Kết quả là `history_length` nhảy lộn xộn, không dự đoán được, và agent rơi vào trạng thái "mất trí nhớ ngẫu nhiên", làm hỏng trải nghiệm người dùng.

---

### Câu 10 — Deploy thật (CP5)

Ghi lại **một** lỗi bạn gặp khi deploy lên cloud (build fail, health check
timeout, sai REDIS_URL, app không đọc `$PORT`...): thông báo lỗi là gì, bạn
tìm ra nguyên nhân bằng cách nào, và sửa ra sao?

- **Lỗi gặp phải:** **Health check timeout do ứng dụng không đọc biến môi trường `$PORT`**.
- **Thông báo lỗi trên Cloud Dashboard (Render/Railway):**
  `Timed out waiting for container to become healthy. Service failed to bind to port within 300 seconds.`
- **Cách tìm ra nguyên nhân:**
  Kiểm tra logs triển khai trong tab Logs của dashboard, thấy uvicorn log dòng: `Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)`. Trong khi đó, nền tảng Cloud chỉ định cổng động qua biến môi trường `$PORT` (ví dụ: `PORT=10000` trên Render hoặc port ngẫu nhiên trên Railway). Vì Dockerfile ban đầu hardcode `--port 8000` trong `CMD`, container chỉ mở cổng 8000, khiến ingress proxy của Cloud không thể kết nối tới cổng được cấp phát dẫn đến health check thất bại và container bị hủy.
- **Cách sửa:**
  Sửa lệnh `CMD` trong `Dockerfile` sang dạng shell-exec nhằm đọc giá trị biến môi trường `$PORT` do Cloud truyền vào (kèm giá trị dự phòng 8000 cho môi trường local):
  ```dockerfile
  CMD ["sh", "-c", "uvicorn app.main:app --host 0.0.0.0 --port ${PORT:-8000}"]
  ```
  Sau khi cập nhật và push lại, ứng dụng lắng nghe chính xác trên cổng được cấp phát, vượt qua health check của Cloud và chuyển sang trạng thái Live.
