# Kế hoạch Rà soát và Gỡ bỏ các Dịch vụ NocoBase Không cần thiết trên VPS

- **Ngày lập:** 18/11/2026
- **Vị trí lưu:** `docs/plans/2026-11-18-ke-hoach-go-nocobase-tren-vps.md`
- **Mục tiêu:** Thu hồi tài nguyên RAM, CPU và Swap trên VPS Dokploy (`217.15.160.118`) bằng cách xác định chính xác và gỡ bỏ các container NocoBase thử nghiệm / thừa thãi, đồng thời bảo vệ 100% dữ liệu của hệ thống Production đang hoạt động.

---

## 1. Hiện trạng Khảo sát thực tế trên Dokploy VPS (`https://dokploy.dsc.edu.vn`)

Qua rà soát chi tiết qua Dokploy API (xem mã nguồn khảo sát tại `scripts/audit_dokploy_full.py`), hiện tại có các dịch vụ liên quan đến NocoBase:

| STT | Dự án (Project) | Tên Dịch vụ | Loại & ID Dokploy | Trạng thái | Domain | Vai trò thực tế & Đánh giá |
| :---: | :--- | :--- | :--- | :---: | :--- | :--- |
| **1** | `nocobase-dsc` | `nocobase` | Compose: `1blJttsJvFU9gsc6hIqnj` | **running** (`done`) | `nocobase.dsc.edu.vn` | **PRODUCTION CHÍNH:** Chứa dữ liệu vận hành học viên, phụ huynh, lớp học (đã gán 1.5GB RAM, 1.5 CPU). **TUYỆT ĐỐI KHÔNG GỠ.** |
| **2** | `duongsinh-studio`| `studio` | Compose: `ku1CZ2aRwn5CAHteBwB76` | **running** (`done`) | `studio.dsc.edu.vn` | **Studio CE:** Nền tảng điều phối agent & quản lý dự án (1.5GB RAM, 1.5 CPU). **GIỮ NGUYÊN.** |
| **3** | `duongsinh-studio`| `nb3-nocolms` | Compose: `EUUMXWbjkhU69t2zI6TOr` | **running** (`done`) | Không domain riêng (gắn nhãn Traefik `nocolms.dsc.edu.vn`) | **THỬ NGHIỆM THỪA:** Một app NocoBase 3 (LMS test) do Studio deploy thử nghiệm trước đây, gồm 1 Postgres (384MB) + 1 Node app (1GB RAM) + 1 init job. Đang chiếm gần **1.4 GB RAM** và CPU. |
| **4** | `nocobase-dsc` | `nococrm` | Compose: `fm37nElqb0l8g_MszHpwG` | **running** (`done`) | `nococrm.dsc.edu.vn` | **NocoCRM (chưa hoàn thiện frontend):** Container chạy backend trả về 404 cho root `/`, chỉ trả `{"ok":true}` ở `/main/api/healthz`. Đang chiếm tài nguyên. |
| **5** | `nocoedu` | `nocoedu` | Compose: `3JwSzK5w1nmVe_dc3q_oi` | **running** (`done`) | `nocoedu.dsc.edu.vn` | **NocoEdu (thử nghiệm):** Build từ repo nocoedu.git, gồm 1 Postgres (384MB) + 1 Node app (1.5GB RAM). Chưa có traffic chính thức. |
| **6** | `nocobase-dsc` | `nocoreach` | App: `elCxX0rd8fx-osA3hhjY-` | **error** | `nocoreach.dsc.edu.vn` | Đang ở trạng thái `error`, không chạy được nhưng vẫn còn cấu hình. |

---

## 2. Chiến lược Gỡ bỏ & Thu hồi Tài nguyên

1. **Nguyên tắc an toàn tối thượng:**
   - Bảo toàn nguyên vẹn 100% dịch vụ Production `nocobase` (`nocobase.dsc.edu.vn`) và `studio` (`studio.dsc.edu.vn`).
   - Sao lưu cấu hình Compose YAML và biến môi trường (Environment variables) của tất cả các service trước khi thao tác.
   - Ưu tiên dừng (`stop`) dịch vụ trước để kiểm tra tính ổn định của hệ thống trước khi xóa hẳn (`delete`).

2. **Dịch vụ ĐỀ XUẤT GỠ BỎ NGAY:**
   - **`nb3-nocolms` (ID: `EUUMXWbjkhU69t2zI6TOr`):** Đây chính là container NocoBase 3 thử nghiệm thừa nhất, đang chiếm tới ~1.4 GB RAM và 1.5 CPU trên VPS mà không có người dùng thật.
   - **`nocoreach` (ID: `elCxX0rd8fx-osA3hhjY-`):** App bị lỗi `error` tồn đọng.

3. **Dịch vụ XIN Ý KIẾN CHỦ DỰ ÁN:**
   - **`nococrm` (ID: `fm37nElqb0l8g_MszHpwG`):** Bạn có đang dùng `nococrm.dsc.edu.vn` không, hay muốn tạm dừng để giải phóng 1GB RAM?
   - **`nocoedu` (ID: `3JwSzK5w1nmVe_dc3q_oi`):** Bạn có đang dùng `nocoedu.dsc.edu.vn` không, hay muốn tạm dừng để giải phóng ~1.8GB RAM (app + db)?

---

## 3. Các bước triển khai thực hiện

### Bước 1: Sao lưu cấu hình
Tự động lưu toàn bộ nội dung `composeFile` và `env` của `nb3-nocolms`, `nocoreach` vào file sao lưu cục bộ `docs/plans/backups/`.

### Bước 2: Dừng (Stop) dịch vụ `nb3-nocolms` và dọn container
Gọi Dokploy API `compose.stop` cho `nb3-nocolms` (`EUUMXWbjkhU69t2zI6TOr`).

### Bước 3: Xóa hoặc Dừng dịch vụ theo xác nhận của bạn
Gỡ bỏ hoàn toàn `nb3-nocolms` hoặc chuyển sang trạng thái dừng để giải phóng RAM ngay lập tức.

### Bước 4: Kiểm tra lại tài nguyên và tình trạng Production
Kiểm tra `https://nocobase.dsc.edu.vn` và `https://studio.dsc.edu.vn` để đảm bảo hoạt động 100% bình thường.

---

## 4. Kết quả Thực hiện (Execution Summary)

*(Sẽ cập nhật sau khi triển khai)*
