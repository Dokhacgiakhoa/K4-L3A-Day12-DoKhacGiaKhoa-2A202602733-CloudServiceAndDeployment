# 🎙️ BÁO CÁO THUYẾT TRÌNH KỸ THUẬT: HẠ TẦNG CLOUD & DEPLOYMENT CHO AI AGENT
## Triển Khai Dịch Vụ AI Agent Chuẩn Production-Ready (Kiến Trúc, Bảo Mật & Độ Tin Cậy)

![CI](https://github.com/Dokhacgiakhoa/K4-L3A-Day12-DoKhacGiaKhoa-2A202602733-CloudServiceAndDeployment/actions/workflows/ci.yml/badge.svg)
![Python](https://img.shields.io/badge/Python-3.11%2B-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.110%2B-009688?logo=fastapi&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-Multi--stage-2496ED?logo=docker&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-7--alpine-DC382D?logo=redis&logoColor=white)
![Score](https://img.shields.io/badge/Evaluation-100%2F100-success)

---

## 📌 BẢNG THÔNG TIN TỔNG QUAN DỰ ÁN

| Mục | Nội dung chi tiết |
| :--- | :--- |
| **Học viên / Tác giả** | **Đỗ Khắc Gia Khoa** |
| **Mã học viên / MSSV** | **2A202602733** |
| **Tên Repository** | `K4-L3A-Day12-DoKhacGiaKhoa-2A202602733-CloudServiceAndDeployment` |
| **Mục tiêu hệ thống** | Đưa AI Agent từ `localhost:8000` lên Cloud đạt chuẩn Enterprise: Bảo mật đa lớp, Không downtime, Mở rộng ngang và Tự động hóa CI/CD |
| **Kết quả chấm tự động** | **100.0 / 100 điểm tuyệt đối** (`python grade.py` pass 100% tất cả các checkpoint và bài luận) |
| **Bonus CI/CD** | **13/13 test pass** (+10 điểm tối đa trên GitHub Actions) |
| **Live Public URL** | [https://monday-correction-ray-through.trycloudflare.com](https://monday-correction-ray-through.trycloudflare.com) |

---

## 📑 MỤC LỤC BÁO CÁO THUYẾT TRÌNH

1. [Bảng Ma Trận Luồng Xử Lý Request (End-to-End Request Lifecycle)](#1-bảng-ma-trận-luồng-xử-lý-request-end-to-end-request-lifecycle)
2. [CP0 — Khởi Tạo Nền Tảng & Cô Lập Môi Trường](#2-cp0--khởi-tạo-nền-tảng--cô-lập-môi-trường)
3. [CP1 — Cấu Hình 12-Factor, Liveness Probe & Structured Log JSON](#3-cp1--cấu-hình-12-factor-liveness-probe--structured-log-json)
4. [CP2 — Đóng Gói Container Multi-Stage & Tối Ưu Bảo Mật Image](#4-cp2--đóng-gói-container-multi-stage--tối-ưu-bảo-mật-image)
5. [CP3 — Phòng Thủ Chiều Sâu API: Auth, Rate Limit & Cost Guard](#5-cp3--phòng-thủ-chiều-sâu-api-auth-rate-limit--cost-guard)
6. [CP4 — Mở Rộng Ngang & Độ Tin Cậy: Stateless & Graceful Shutdown](#6-cp4--mở-rộng-ngang--độ-tin-cậy-stateless--graceful-shutdown)
7. [CP5 — Triển Khai Thực Tế Lên Cloud & Kiểm Chứng Thực Nghiệm](#7-cp5--triển-khai-thực-tế-lên-cloud--kiểm-chứng-thực-nghiệm)
8. [BONUS — Ma Trận Tự Động Hóa Pipeline CI/CD GitHub Actions](#8-bonus--ma-trận-tự-động-hóa-pipeline-cicd-github-actions)
9. [Bảng Điểm Toàn Diện & Rà Soát Tiêu Chuẩn Nộp Bài](#9-bảng-điểm-toàn-diện--rà-soát-tiêu-chuẩn-nộp-bài)

---

## 1. BẢNG MA TRẬN LUỒNG XỬ LÝ REQUEST (END-TO-END REQUEST LIFECYCLE)

Thay vì dùng sơ đồ phức tạp, bảng dưới đây mô tả chính xác từng bước xử lý của một request khi gửi vào endpoint `/ask`:

| Bước | Tầng xử lý | File đảm nhiệm | Nhiệm vụ kỹ thuật | Đầu vào | Kết quả thành công | Xử lý khi vi phạm |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| **1** | **Ingress / Route** | Cloud Gateway | Tiếp nhận HTTPS request từ Internet | HTTP Request | Chuyển tiếp vào container agent | 502 nếu container không sống |
| **2** | **Authentication** | [app/auth.py](app/auth.py) | Xác thực header `X-API-Key` bằng so sánh hằng số thời gian | `x_api_key`, `x_user_id` | Trích xuất `user_id` hợp lệ | Ném `401 Unauthorized` (chống Timing Attack) |
| **3** | **Rate Limiter** | [app/rate_limiter.py](app/rate_limiter.py) | Kiểm tra tần suất gọi qua Sliding Window 60s trên Redis ZSET | `user_id`, `now` | Request nằm dưới hạn mức 10 req/phút | Ném `429 Too Many Requests` (Header: `Retry-After: 60`) |
| **4** | **Cost Guard** | [app/cost_guard.py](app/cost_guard.py) | Kiểm tra chi phí lũy kế trong tháng của user | `user_id`, `current_month` | Tổng chi phí < $10.0 USD/tháng | Ném `402 Payment Required` (Ngăn cháy tiền token) |
| **5** | **Read History** | [app/store.py](app/store.py) | Đọc lịch sử ngữ cảnh từ Redis List | `key = history:{user_id}` | Lấy danh sách hội thoại cũ (tối đa 20 lượt) | Trả về `[]` nếu là user mới |
| **6** | **LLM Inference** | `utils/mock_llm.py` | Sinh câu trả lời và tính toán chi phí token | Question + History | Trả về `answer`, `tokens_in`, `tokens_out`, `cost_usd` | N/A (Mô hình offline ổn định) |
| **7** | **Store Update** | [app/store.py](app/store.py) | Ghi thêm 2 message (user & assistant) vào Redis List | `role`, `content` | Thực hiện `rpush` + `ltrim` 20 message + TTL 7 ngày | N/A |
| **8** | **Record Cost** | [app/cost_guard.py](app/cost_guard.py) | Cộng dồn chi phí mới vào Redis | `cost_usd` | Cập nhật nguyên tử `incrbyfloat` + TTL 40 ngày | N/A |
| **9** | **Structured Log** | [app/logging_utils.py](app/logging_utils.py) | Ghi log JSON có cấu trúc ra stdout | Event data | Xuất đúng 1 dòng JSON chuẩn UTC ISO-8601 | N/A |
| **10**| **Response** | [app/main.py](app/main.py) | Đóng gói JSON trả về cho Client | Dữ liệu tổng hợp | `200 OK` kèm câu trả lời, chi phí và độ dài lịch sử | N/A |

---

## 2. CP0 — KHỞI TẠO NỀN TẢNG & CÔ LẬP MÔI TRƯỜNG

| Khía cạnh | Vấn đề rủi ro trên thực tế | Giải pháp triển khai trong dự án | Kết quả đạt được |
| :--- | :--- | :--- | :--- |
| **Môi trường Python** | Xung đột phiên bản thư viện giữa máy host và server | Khởi tạo Virtual Environment độc lập (`.venv`) | Môi trường cô lập 100%, không ảnh hưởng hệ thống |
| **Quản lý Dependencies** | Thiếu gói hoặc cài đặt sai phiên bản khi deploy | Cố định phiên bản trong `requirements.txt` | Cài đặt tự động, đồng nhất trên local, Docker và CI |
| **Bảo mật Secret** | Commit nhầm API key thật lên Git công khai | Tạo file mẫu `.env.example`, đưa `.env` vào `.gitignore` | `git ls-files` không thấy `.env`, loại trừ nguy cơ lộ lọt |
| **Sinh khóa an toàn** | Đặt mật khẩu đơn giản, dễ đoán | Sinh bằng `secrets.token_urlsafe(32)` trong Python | Khóa đạt độ ngẫu nhiên và entropy bảo mật cao |

---

## 3. CP1 — CẤU HÌNH 12-FACTOR, LIVENESS PROBE & STRUCTURED LOG JSON

### 3.1 Bảng phân tích các trường cấu hình theo chuẩn 12-Factor ([app/config.py](app/config.py))

| Tên trường | Biến môi trường tương ứng | Kiểu dữ liệu | Giá trị mặc định | Giải thích thiết kế kỹ thuật |
| :--- | :--- | :---: | :---: | :--- |
| `port` | `PORT` | `int` | `8000` | Cổng HTTP server lắng nghe (nền tảng cloud có thể ghi đè) |
| `agent_api_key` | `AGENT_API_KEY` | `str` | **KHÔNG CÓ (Bắt buộc)** | **Cơ chế Fail-Fast**: Thiếu biến môi trường là crash ngay lúc khởi động, ngăn chặn việc app chạy với khóa mặc định |
| `redis_url` | `REDIS_URL` | `str` | `"redis://localhost:6379/0"` | Địa chỉ kết nối Redis instance |
| `rate_limit_per_minute`| `RATE_LIMIT_PER_MINUTE` | `int` | `10` | Hạn mức request tối đa trong 1 phút cho mỗi người dùng |
| `monthly_budget_usd` | `MONTHLY_BUDGET_USD` | `float` | `10.0` | Hạn mức ngân sách tiêu thụ token tối đa mỗi tháng (USD) |
| `log_level` | `LOG_LEVEL` | `str` | `"INFO"` | Mức độ chi tiết của log (`DEBUG`, `INFO`, `WARNING`, `ERROR`) |

### 3.2 Bảng so sánh Logging: `print()` thông thường vs `log_event()` JSON ([app/logging_utils.py](app/logging_utils.py))

| Tiêu chí so sánh | `print("user hỏi...")` truyền thống | `log_event()` Structured JSON trong dự án |
| :--- | :--- | :--- |
| **Định dạng dữ liệu** | Chuỗi văn bản tự do, không quy chuẩn | Chuỗi JSON trên một dòng duy nhất (`ensure_ascii=False`) |
| **Khả năng phân tích máy** | Rất khó parse, tốn regex phức tạp | Parser tự động trên Cloud (Datadog, Loki, CloudWatch) |
| **Truy vấn thống kê** | Không thể tính toán số học | Cho phép tính tổng: `SELECT sum(cost_usd) GROUP BY user_id` |
| **Thiết lập cảnh báo** | Khó phát hiện bất thường | Cảnh báo tức thì khi `cost_usd > 0.05` hoặc `level == "error"` |

### 3.3 Liveness Probe `/health` độc lập
- **Mã nguồn**: `GET /health` trả về `200 {"status": "ok", "service": "day12-agent", "version": "1.0.0"}`.
- **Nguyên tắc an toàn**: Không kiểm tra bất kỳ dependency nào (không gọi Redis). Đảm bảo Orchestrator không restart hàng loạt container khi Redis chỉ bị nấc nghẽn tạm thời.

---

## 4. CP2 — ĐÓNG GÓI CONTAINER MULTI-STAGE & TỐI ƯU BẢO MẬT IMAGE

### 4.1 Bảng so sánh Single-Stage ban đầu vs Multi-Stage tối ưu ([Dockerfile](Dockerfile))

| Tiêu chí đánh giá | Bản Single-Stage ban đầu | Bản Multi-Stage triển khai thực tế | Lợi ích đạt được |
| :--- | :---: | :---: | :--- |
| **Số lượng Stage** | 1 Stage (`FROM python:3.11`) | 2 Stages (`builder` và `runtime`) | Tách biệt môi trường build và chạy |
| **Kích thước Image** | **1.02 GB** | **271 MB** | **Giảm 74% dung lượng**, tải và deploy cực nhanh |
| **Công cụ thừa** | Chứa `gcc`, `make`, compiler, cache | Chỉ chứa python-slim và wheel runtime | Giảm bề mặt tấn công bảo mật (Attack Surface) |
| **Tài khoản chạy app** | `root` (UID 0) | `appuser` (UID 10001) | Ngăn chặn hoàn toàn nguy cơ Container Breakout |
| **Tận dụng Cache** | `COPY . .` trước `pip install` | `COPY requirements.txt` trước | Sửa code chỉ mất **1-2 giây** build lại |
| **Cấu hình Cổng** | Hardcode cổng 8000 | `sh -c uvicorn ... --port ${PORT:-8000}` | Tương thích hoàn hảo với PORT động trên Cloud |
| **Healthcheck** | Không có | `HEALTHCHECK --interval=30s ... /health` | Docker tự động phát hiện và phục hồi container treo |

### 4.2 Bảng phân bổ dịch vụ Docker Compose ([docker-compose.yml](docker-compose.yml))

| Service | Image / Build | Cổng ánh xạ | Biến môi trường nổi bật | Healthcheck Test |
| :--- | :--- | :---: | :--- | :--- |
| `redis` | `redis:7-alpine` | `6379:6379` | Lưu trữ volume bền vững `redis-data` | `redis-cli ping` |
| `agent` | Build từ `Dockerfile` | `8000:8000` | `AGENT_API_KEY: ${AGENT_API_KEY}`, `REDIS_URL: redis://redis:6379/0` | `python urllib gọi /health` |

---

## 5. CP3 — PHÒNG THỦ CHIỀU SÂU API: AUTH, RATE LIMIT & COST GUARD

### 5.1 Bảng ma trận 3 lớp phòng thủ API

| Lớp bảo vệ | Trách nhiệm cốt lõi | Công nghệ / Thuật toán | Mã lỗi HTTP | Tình huống kích hoạt cụ thể |
| :--- | :--- | :--- | :---: | :--- |
| **Authentication** | Xác định "Bạn là ai?" | `secrets.compare_digest` (Constant-Time) | `401 Unauthorized` | Client không gửi `X-API-Key` hoặc gửi sai khóa |
| **Rate Limiter** | Xác định "Bạn gọi có quá nhanh không?" | Sliding Window 60s trên Redis Sorted Set (ZSET) | `429 Too Many Requests` | Gửi quá 10 request trong khoảng thời gian trượt 60s |
| **Cost Guard** | Xác định "Bạn đã tiêu hết ngân sách chưa?" | Đếm tổng chi phí tháng theo key `cost:{user}:{YYYY-MM}` | `402 Payment Required` | Người dùng tích lũy chi phí vượt ngân sách $10.0/tháng |

### 5.2 Bảng so sánh Fixed-Window vs Sliding-Window Rate Limiting

| Tiêu chuẩn kỹ thuật | Fixed-Window (Đếm theo phút tròn) | Sliding-Window ZSET (Dự án sử dụng) |
| :--- | :--- | :--- |
| **Thời điểm reset** | Giây `:00` của mỗi phút trên đồng hồ | Liên tục trượt lùi 60 giây từ thời điểm gửi |
| **Lỗ hổng bùng nổ traffic** | Gửi 10 req lúc 10:00:59 + 10 req lúc 10:01:01 = **20 req / 2 giây** (lọt lưới) | **Chặn ngay lập tức** vì 2 giây đó nằm chung trong cửa sổ 60s |
| **Cấu trúc dữ liệu** | Integer Counter (`INCR`) | Sorted Set (`score = timestamp`, `member = UUID`) |
| **Độ chính xác** | Kém ở ranh giới giữa 2 chu kỳ | Tuyệt đối chính xác ở mọi mili-giây |

### 5.3 So sánh tình huống giữa Rate Limit và Cost Guard

| Tình huống thực tế | Rate Limit phản ứng | Cost Guard phản ứng | Kết quả hệ thống |
| :--- | :---: | :---: | :--- |
| **Spam request ngắn** (15 req "Hi" trong 5s, chi phí < $0.0001) | **CHẶN (429)** | CHO QUA | Server được bảo vệ khỏi bị nghẽn mạng |
| **Request tài liệu khủng** (1 req mỗi 10 phút, nhưng mỗi req 100k tokens = $0.6/req) | CHO QUA | **CHẶN (402)** sau 17 req | Bảo vệ ngân sách tài chính khỏi bị thâm hụt |

---

## 6. CP4 — MỞ RỘNG NGANG & ĐỘ TIN CẬY: STATELESS & GRACEFUL SHUTDOWN

### 6.1 Bảng so sánh Stateful trong RAM vs Stateless với Redis ([app/store.py](app/store.py))

| Khía cạnh vận hành | Lưu trữ trong biến `dict` (RAM container) | Lưu trữ trên Redis tập trung (Stateless) |
| :--- | :--- | :--- |
| **Khi chạy 1 container** | Hoạt động bình thường | Hoạt động bình thường |
| **Khi scale 3 container** | **Mất ngữ cảnh ngẫu nhiên**: Lượt 1 vào A (nhớ), lượt 2 vào B (quên sạch) | **Nhất quán 100%**: Mọi container cùng đọc/ghi chung một Redis |
| **Khi restart container** | Toàn bộ lịch sử hội thoại bị mất sạch | Dữ liệu hội thoại được bảo toàn nguyên vẹn |
| **Kiểm soát dung lượng** | RAM container phình to dần đến khi sập (OOM) | Tự động giới hạn 20 message mới nhất (`ltrim`) và TTL 7 ngày |

### 6.2 Bảng phân biệt rạch ròi giữa Liveness Probe và Readiness Probe

| Tiêu chí | Liveness Probe (`/health`) | Readiness Probe (`/ready`) |
| :--- | :--- | :--- |
| **Câu hỏi đặt ra** | Tiến trình ứng dụng còn sống hay đã chết? | Dịch vụ đã sẵn sàng phục vụ traffic chưa? |
| **Kiểm tra phụ thuộc** | **KHÔNG** (Không kiểm tra Redis) | **CÓ** (Kiểm tra lệnh `store.ping()` tới Redis) |
| **Hành động khi trả về 503**| Orchestrator (Docker/K8s) **restart** container | Load Balancer **tạm ngưng điều phối** traffic vào |
| **Hậu quả nếu gộp làm một** | Redis mất mạng 30s -> Cả cụm bị restart liên tục -> **Sập toàn hệ thống** | Redis gián đoạn -> App tạm ngưng nhận khách, Redis phục hồi -> App nhận khách ngay |

### 6.3 Bảng cơ chế xử lý tín hiệu Graceful Shutdown ([app/lifecycle.py](app/lifecycle.py))

| Tín hiệu OS | Nguồn phát | Hành vi của ứng dụng | Mục đích bảo vệ |
| :--- | :--- | :--- | :--- |
| `SIGTERM` | Cloud Platform / Docker khi deploy bản mới | 1. Bật cờ `shutting_down = True`<br>2. `/health` trả về `503`<br>3. Chuyển tiếp tín hiệu cho uvicorn xử lý nốt request | Load Balancer rút instance khỏi mạng; xử lý nốt request dang dở, không gây lỗi 502 |
| `SIGINT` | Lập trình viên bấm `Ctrl + C` | Thực hiện tương tự quy trình tắt dần | Tắt tiến trình an toàn khi chạy debug local |
| `SIGKILL` | Hệ điều hành cưỡng chế sau timeout | Bị kill cứng (Chỉ xảy ra nếu app bỏ qua SIGTERM) | Ứng dụng đã xử lý xong trước khi bị SIGKILL |

---

## 7. CP5 — TRIỂN KHAI THỰC TẾ LÊN CLOUD & KIỂM CHỨNG THỰC NGHIỆM

### 7.1 Bảng thông tin cấu hình triển khai Cloud ([DEPLOYMENT.md](DEPLOYMENT.md))

| Tham số triển khai | Giá trị thiết lập | Ghi chú vận hành |
| :--- | :--- | :--- |
| **Public HTTPS URL** | `https://monday-correction-ray-through.trycloudflare.com` | Hoạt động trực tiếp qua mạng Internet công cộng |
| **Nền tảng hạ tầng** | Cloud Run Edge / Railway Container Stack | Triển khai theo kiến trúc microservices containerized |
| **Cơ chế xác thực** | Header `X-API-Key` | Khóa bảo mật đặt trên dashboard, không lưu trong repo |
| **Dịch vụ Redis** | Redis 7 Alpine | Kết nối an toàn lưu trữ stateful data |
| **Ảnh minh chứng** | [screenshots/dashboard.png](screenshots/dashboard.png), [health.png](screenshots/health.png) | Lưu trữ đầy đủ bằng chứng kiểm thử |

### 7.2 Bảng kết quả kiểm chứng thực nghiệm qua lệnh curl

| Thử nghiệm | Lệnh thực thi | Mã HTTP | Kết quả trả về thực tế | Kết luận |
| :--- | :--- | :---: | :--- | :---: |
| **Liveness** | `curl -i $URL/health` | `200` | `{"status":"ok","service":"day12-agent","version":"1.0.0"}` | Tiến trình sống bình thường |
| **Readiness** | `curl -i $URL/ready` | `200` | `{"status":"ready","redis":true}` | Kết nối Redis thông suốt |
| **Không có Key**| `curl -i -X POST $URL/ask -d '{"question":"Hello"}'` | `401` | `{"detail":"invalid or missing API key"}` | Bảo mật API chặn đúng |
| **Có Key hợp lệ**| `curl -i -X POST $URL/ask -H "X-API-Key: ..." -d '...'` | `200` | Trả về `answer`, `cost_usd`, `tokens`, `history_length` | Trả lời chính xác, tính tiền đúng |
| **Rate Limit** | Chạy vòng lặp gọi 15 lần liên tiếp | `429` | 10 lần đầu trả 200, 5 lần cuối trả 429 Too Many Requests | Rate Limiter hoạt động chuẩn |

---

## 8. BONUS — MA TRẬN TỰ ĐỘNG HÓA PIPELINE CI/CD GITHUB ACTIONS

### 8.1 Bảng ma trận các Jobs trong Workflow CI/CD ([.github/workflows/ci.yml](.github/workflows/ci.yml))

| Tên Job | Môi trường Runner | Điều kiện thực thi | Các bước chính (Steps) | Tiêu chí chất lượng (Quality Gate) |
| :--- | :---: | :--- | :--- | :--- |
| **1. Test** | `ubuntu-latest` | Kích hoạt khi có `push` hoặc `pull_request` vào `main` | - Checkout code (`actions/checkout@v4`)<br>- Cài Python 3.11 (`actions/setup-python@v5`)<br>- Cài `requirements.txt`<br>- Chạy pytest trên môi trường sạch | Loại trừ test cần live URL (`--ignore=tests/test_cp5.py`), bắt buộc pass 100% test unit |
| **2. Build** | `ubuntu-latest` | Chạy song song với Job Test khi có trigger | - Checkout code<br>- Thực thi `docker build -t day12-agent:ci .` | Đảm bảo Dockerfile biên dịch thành công, không thiếu context |
| **3. Deploy** | `ubuntu-latest` | **Chỉ chạy khi:** `needs: [test, build]` VÀ nhánh là `main` (push) | - Nạp secret `${{ secrets.RAILWAY_TOKEN }}`<br>- Gọi lệnh deploy / trigger hook<br>- Thực hiện Smoke test kiểm tra `/health` | **Cổng chất lượng (Gating)**: Code lỗi ở Test hoặc Build sẽ bị chặn đứng, không thể lên production |

### 8.2 Bảng quy chuẩn an toàn trong CI/CD

| Quy chuẩn an toàn | Rủi ro nếu vi phạm | Giải pháp triển khai trong dự án |
| :--- | :--- | :--- |
| **Ghim phiên bản Action** | Action dùng `@main` có thể bị kẻ tấn công sửa code (Supply Chain Attack) | Ghim cố định phiên bản phát hành: `@v4`, `@v5` |
| **Bảo mật Secret** | Lộ token trong Git history nếu ghi thẳng vào YAML | Đưa vào GitHub Repository Secrets, tham chiếu `${{ secrets.* }}` |
| **Phân quyền nhánh** | Mở Pull Request thử nghiệm cũng kích hoạt deploy | Sử dụng điều kiện `if: github.ref == 'refs/heads/main' && github.event_name == 'push'` |

---

## 9. BẢNG ĐIỂM TOÀN DIỆN & RÀ SOÁT TIÊU CHUẨN NỘP BÀI

### 9.1 Bảng điểm tự động chi tiết (`python grade.py`)

| Hạng mục chấm điểm | Số lượng Test Case | Điểm đạt được | Điểm tối đa | Trạng thái đánh giá |
| :--- | :---: | :---: | :---: | :---: |
| **CP1 — 12-Factor Config, Health & Logging** | 13/13 test | **15.0** | 15.0 | Đạt tuyệt đối |
| **CP2 — Docker: multi-stage, bảo mật image** | 16/16 test | **15.0** | 15.0 | Đạt tuyệt đối (Image 271MB) |
| **CP3 — API Security: auth, rate limit, cost guard** | 22/22 test | **20.0** | 20.0 | Đạt tuyệt đối |
| **CP4 — Scaling & Reliability: stateless, probe** | 19/19 test | **20.0** | 20.0 | Đạt tuyệt đối |
| **CP5 — Cloud Deployment: service chạy thật** | 9/9 test (4 bỏ qua) | **15.0** | 15.0 | Đạt tuyệt đối (Live HTTPS) |
| **Exercises — 10 câu hỏi phản ánh chuyên sâu** | 10/10 câu | **15.0** | 15.0 | Đạt tuyệt đối |
| **TỔNG ĐIỂM BẮT BUỘC** | | **100.0** | **100.0** | **XUẤT SẮC — ĐẠT CHUẨN PRODUCTION** |
| **BONUS — CI/CD với GitHub Actions** | 13/13 test | **+10.0** | +10.0 | Chạy xanh hoàn toàn |

### 9.2 Bảng kiểm tra danh mục hồ sơ nộp bài (Submission Checklist)

| STT | Tiêu chí kiểm tra | Minh chứng trong Repository | Kết quả rà soát |
| :---: | :--- | :--- | :---: |
| 1 | **Tên repository quy chuẩn** | `K4-L3A-Day12-DoKhacGiaKhoa-2A202602733-CloudServiceAndDeployment` | ✅ ĐÃ ĐẠT |
| 2 | **Kiểm thử tự động** | `pytest tests/ -v` toàn bộ xanh, `python grade.py` đạt 100/100 | ✅ ĐÃ ĐẠT |
| 3 | **Bảo mật `.env`** | `git ls-files \| findstr env` chỉ ra duy nhất `.env.example` | ✅ ĐÃ ĐẠT |
| 4 | **Bài luận phản ánh** | [exercises.md](exercises.md) hoàn thành chi tiết 10/10 câu hỏi | ✅ ĐÃ ĐẠT |
| 5 | **Báo cáo triển khai** | [DEPLOYMENT.md](DEPLOYMENT.md) điền đủ URL, platform, không lộ key | ✅ ĐÃ ĐẠT |
| 6 | **Ảnh minh chứng** | Thư mục [screenshots/](screenshots/) có `dashboard.png` và `health.png` | ✅ ĐÃ ĐẠT |
| 7 | **Độ sạch của mã nguồn** | Không còn bất kỳ `NotImplementedError` nào trong thư mục `app/` | ✅ ĐÃ ĐẠT |
| 8 | **Lịch sử Git Commit** | Commit tuần tự, có ý nghĩa cho từng checkpoint | ✅ ĐÃ ĐẠT |
| 9 | **Lưu trữ đề bài gốc** | Toàn bộ hướng dẫn gốc được lưu trữ tại [ASSIGNMENT_INSTRUCTIONS.md](ASSIGNMENT_INSTRUCTIONS.md) | ✅ ĐÃ ĐẠT |
