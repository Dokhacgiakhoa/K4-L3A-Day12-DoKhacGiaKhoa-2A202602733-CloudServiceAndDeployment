# K4 — Level 3A, Ngày 12: Hạ Tầng Cloud & Deployment (240 phút)

> **Tài liệu hướng dẫn gốc của bài lab.**
> Học viên: **Đỗ Khắc Gia Khoa** (MSSV: `2A202602733`)

---

## ⚠️ Bài Làm Cá Nhân

**Đây là bài tập cá nhân. Mỗi học viên nộp một repository của riêng mình.**

Tài liệu chính thức của bài lab:
- [SUBMISSION.md](SUBMISSION.md) — cấu trúc bài nộp, tên repo và nơi nộp
- [RUBRIC.md](RUBRIC.md) — tiêu chí chấm, bằng chứng và điều kiện mất điểm
- [CHECKPOINTS.md](CHECKPOINTS.md) — sản phẩm, kiến thức và cách tự kiểm tra từng checkpoint
- [RULES.md](RULES.md) — quy định làm bài, dùng AI, hợp tác và bảo mật

| Được phép | Không được phép |
|-----------|-----------------|
| Đọc tài liệu, Stack Overflow, tra AI để hiểu khái niệm | Sao chép code của học viên khác |
| Hỏi Lab Coach khi bị kẹt | Dùng chung repo, chung commit history |
| Thảo luận **cách tiếp cận** với bạn cùng lớp | Nhờ người khác làm hộ, kể cả một phần |
| Dùng AI để giải thích lỗi | Nộp code mà bạn không giải thích được |

---

## 📦 Cách Đặt Tên Repository

Repo nộp bài **bắt buộc** đặt tên theo mẫu:
```
K4-L3A-DAY12-<HoVaTen>-<MSSV>-<TenBai>
```
Repo hiện tại: `K4-L3A-Day12-DoKhacGiaKhoa-2A202602733-CloudServiceAndDeployment`

---

## Mục Tiêu & Lịch Trình Checkpoint

| Checkpoint | Nội dung | File kiểm thử | Điểm |
| :--- | :--- | :--- | :---: |
| **CP1** | 12-Factor Config, Health, Logging | `tests/test_cp1.py` | 15 |
| **CP2** | Docker: multi-stage, bảo mật image | `tests/test_cp2.py` | 15 |
| **CP3** | API Security: auth, rate limit, cost guard | `tests/test_cp3.py` | 20 |
| **CP4** | Scaling & Reliability | `tests/test_cp4.py` | 20 |
| **CP5** | Deploy lên cloud | `tests/test_cp5.py` | 15 |
| **Exercises** | Hoàn thiện 10 câu phản ánh | `exercises.md` | 15 |
| **BONUS** | CI/CD với GitHub Actions | `tests/test_bonus_cicd.py` | +10 |

Chi tiết từng bước triển khai: xem [LAB_GUIDE.md](LAB_GUIDE.md).
