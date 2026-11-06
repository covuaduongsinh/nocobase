# Kế hoạch chi tiết bước 2.5 — Webhook SePay: tự ghi nhận học phí chuyển khoản

Thuộc kế hoạch giai đoạn 2 `docs/plans/2026-10-31-giai-doan-2-cong-phu-huynh-zalo-vietqr.md` (mục 3.3, bước 2.5). Lập ngày 01/11/2026.

## 1. Rà soát tiến độ (01/11/2026)

- 2.1, 2.2 và 2.3 đã xong (xem mục Tiến độ của kế hoạch chiến lược).
- 2.4 Zalo ZNS đã xong phần mã. Commit `f69b6d5`, `312cfec`, `0b7fcb0` (repo `duongsinh-nocobase`). Verify trên production đạt 157/0. Hệ thống đang ở chế độ `tat` và chờ chủ dự án gửi App ID/secret, mẫu ZNS và refresh token.
- **2.5 là bước kế tiếp.** Phần mã làm được ngay bằng payload mẫu. Chỉ khi chạy thật mới cần tài khoản SePay và khoá API.

Đã chốt (31/10/2026):

- Cổng: **SePay**.
- Tài khoản nhận: **VPBank (BIN 970432) 168686838, DUONG SINH CHESS JSC**.
- Nội dung chuyển khoản do QR sinh ra: `DSC <mã đăng ký>`. Mã đăng ký có dạng `DK` + 5 chữ số, vd `DSC DK00352` (xem `setup/ui/vietqr.mjs`).

## 2. Mục tiêu

Phụ huynh quét QR và chuyển khoản. SePay gọi webhook về NocoBase. Hệ thống tự tạo `dsc_phieu_thu` (loại `hoc_phi`, hình thức `vietqr`, gắn gói và học viên). Từ đó các workflow sẵn có tự chạy: **F3** tính lại đã thu và công nợ, **W10** đưa tin ZNS "đã nhận học phí" vào hàng đợi. Giao dịch không nhận diện được thì nằm chờ để quản lý xử lý tay.

Thước đo: ít nhất 80% phiếu thu học phí mới được ghi nhận tự động.

## 3. Thiết kế

### 3.1 Collection `dsc_giao_dich_ngan_hang` (sổ mọi giao dịch nhận về)

| Trường                                 | Ghi chú                                                                 |
| -------------------------------------- | ----------------------------------------------------------------------- |
| `ma_giao_dich_cong`                    | string, **unique**, là `id` giao dịch do SePay gửi. Dùng để chống trùng |
| `ngan_hang`, `so_tai_khoan`            | string, lấy từ `gateway` và `accountNumber`                             |
| `thoi_diem`                            | datetime, lấy từ `transactionDate` (giờ VN)                             |
| `chieu`                                | select `vao` / `ra` (`transferType` in / out)                           |
| `so_tien`                              | money (`transferAmount`)                                                |
| `noi_dung`                             | text (`content`); `ma_tham_chieu` string (`referenceCode`)              |
| `du_lieu_goc`                          | json, toàn bộ payload để tra soát                                       |
| `trang_thai`                           | select `da_ghi_nhan` / `cho_xu_ly` / `bo_qua` / `loi`                   |
| `ly_do`                                | text, giải thích vì sao chờ xử lý, bỏ qua hoặc lỗi                      |
| `phieu_thu`, `dang_ky_goi`, `hoc_vien` | quan hệ m2o, điền khi ghép được                                         |

ACL:

- `dsc_quan_ly` và `dsc_ke_toan` được xem giao dịch. Họ chỉ sửa được `trang_thai`, `ly_do` và ba quan hệ, để ghép tay.
- `dsc_phu_huynh` không có quyền gì. Thêm bảng này vào bộ dò của `setup/parent-isolation.mjs`.

### 3.2 Plugin `@duongsinh/plugin-dsc-bank-webhook` (`D:\code\duongsinh-nocobase\plugins\plugin-dsc-bank-webhook\`)

Chép khung từ `plugins/plugin-dsc-zalo/`. Build và cài bằng `plugins/build.mjs` và `plugins/install.mjs`. Plugin chỉ có phần server.

- **Endpoint** `POST /api/dscBankWebhook:receive`. Mở công khai bằng `this.app.acl.allow('dscBankWebhook', 'receive', 'public')`, rồi tự xác thực bên trong:
  - Header `Authorization: Apikey <key>` (đây là kiểu "API Key" của SePay). So bằng `crypto.timingSafeEqual` với biến môi trường `DSC_SEPAY_API_KEY`.
  - Thiếu biến môi trường thì trả 503 và không ghi gì. Sai khoá thì trả 401 và ghi log cảnh báo, không lưu payload.
  - Plugin `dsc-guard` phải cho resource này đi qua. Kiểm tra lại rằng `parentGuard` và `workflowTriggerGuard` không chặn request ẩn danh.
- **Xử lý** (`src/server/receive.ts`, viết thành hàm thuần tối đa để test được). Tất cả chạy trong **một transaction**:
  1. Kiểm tra payload. Thiếu `id`, `transferAmount` hoặc `transferType` thì trả 400.
  2. **Chống trùng:** nếu đã có `ma_giao_dich_cong = id` thì trả `200 {success:true, duplicate:true}` và không làm gì thêm. SePay sẽ gửi lại khi không nhận được 2xx, nên chống trùng là bắt buộc. Lỗi vi phạm unique khi hai request tới cùng lúc cũng phải trả về 200 trùng.
  3. Lưu giao dịch rồi phân loại:
     - `transferType != 'in'` thì `bo_qua` ("tiền ra").
     - `accountNumber` khác `DSC_BANK_ACCOUNT` (biến môi trường, mặc định `168686838`) thì `bo_qua`.
     - Tách mã gói bằng regex `/DK\s?(\d{5})/i` trên `content`. Ngân hàng hay bỏ dấu cách, vd `DSCDK00352`. Không có mã, có nhiều mã khác nhau, hoặc mã không tồn tại thì `cho_xu_ly` và ghi `ly_do` tương ứng.
     - Gói ở trạng thái `huy` thì `cho_xu_ly`.
     - Ghép được gói thì **tạo `dsc_phieu_thu` bằng repository** để hook và workflow F3, W10 chạy: `so_tien = transferAmount`, `ngay_thu` là ngày của `transactionDate`, `hinh_thuc = 'vietqr'`, `loai_thu = 'hoc_phi'`, `trang_thai = 'da_thu'`, `dangKyGoiId`, `hocVienId` lấy từ gói, `ghi_chu = 'SePay <referenceCode>'`. Sau đó đặt giao dịch `da_ghi_nhan` và gắn `phieu_thu`.
     - Số tiền khác công nợ (thiếu hoặc thừa) **vẫn ghi nhận**. Công thức sẽ tự đưa `tt_thanh_toan` về `mot_phan` hoặc `thua`, và `ly_do` ghi rõ chênh lệch để kế toán xem.
  4. Trả `200 {success: true}` ngay cả khi giao dịch rơi vào `cho_xu_ly`, để SePay không gửi lại. Chỉ trả 5xx khi lỗi hệ thống thật (transaction đã rollback), lúc đó để SePay thử lại.
- **Thông báo:** giao dịch `cho_xu_ly` thì gửi thông báo trong app (kênh `dsc_in_app`) cho `dsc_quan_ly` và `dsc_ke_toan`. Nên làm bằng workflow **W13** (`dsc_w13_gd_cho_xu_ly`, collection `dsc_giao_dich_ngan_hang`, tạo mới, điều kiện `trang_thai = cho_xu_ly`) để giữ đúng mẫu trong `setup/workflows/apply.mjs`.
- **Ghép tay:** trên trang "Giao dịch ngân hàng", quản lý hoặc kế toán chọn gói cho giao dịch `cho_xu_ly`. Có hai cách để tạo phiếu thu sau khi ghép:
  - **Khuyên dùng:** workflow **W14**, nút custom-action "Ghi nhận phiếu thu" trên từng giao dịch. Workflow chạy đồng bộ, chỉ chạy khi `phieu_thuId` còn rỗng, và tạo phiếu thu giống bước 3 ở trên.
  - Cách khác là action của plugin.
  - Với W14, phải thêm vai trò được bấm (`dsc_ke_toan`, `dsc_quan_ly`) vào `WORKFLOW_TRIGGER_ROLES` trong `plugins/plugin-dsc-guard/src/server/guard.ts`. Nếu nút là custom-action gắn bản ghi (type 1), cần kiểm tra guard có áp dụng không.

### 3.3 Giao diện (`setup/ui/pages.mjs`)

- Thêm trang "Giao dịch ngân hàng" vào nhóm Học phí (cho quản lý và kế toán).
  - Bảng sắp mới nhất trước, mặc định lọc `cho_xu_ly`.
  - Cột: thời điểm, số tiền, nội dung, trạng thái, lý do, gói, phiếu thu.
  - Popup sửa có ba quan hệ và nút W14.
- Bảng có hơn 10 trường thì phải khai báo `FIELD_GROUPS` (bẫy validator).
- Bảng điều khiển: thêm một ô thống kê "Giao dịch chờ xử lý" (SQL đếm).

### 3.4 Script và tài liệu

- `setup/sepay.mjs`:
  - `sample <ma_dk> <so_tien> [--id X]`: dựng payload mẫu đúng định dạng SePay và POST tới `NB_URL`, dùng khoá đọc từ biến môi trường. Script dựng JSON trong Node (bẫy UTF-8 của Git Bash).
  - `status`: đếm giao dịch theo trạng thái.
- `docs/huong-dan-sepay.md`:
  - Chủ dự án đăng ký SePay và liên kết VPBank 168686838.
  - Tạo webhook với URL `https://nocobase.dsc.edu.vn/api/dscBankWebhook:receive`, kiểu xác thực **API Key**, chỉ nhận **tiền vào**.
  - Đặt `DSC_SEPAY_API_KEY` trong Dokploy rồi deploy lại.
- **Trước khi viết mã, đối chiếu định dạng payload và kiểu xác thực với tài liệu chính thức SePay** (docs.sepay.vn, mục webhook). Các trường dự kiến: `id`, `gateway`, `transactionDate`, `accountNumber`, `subAccount`, `code`, `content`, `transferType`, `transferAmount`, `accumulated`, `referenceCode`, `description`. Có điểm nào chưa xác minh được thì ghi vào mục "PHẢI xác minh trước khi bật thật", giống cách làm ở 2.4.

## 4. Thứ tự triển khai

1. Đọc file bàn giao, README và các plugin `dsc-zalo`, `dsc-guard`. Chạy `git status` và chỉ bắt đầu khi cây sạch.
2. Viết schema và ACL, chạy `run.mjs collections` và `run.mjs roles` trên dev.
3. Viết plugin, unit test, build và cài trên dev.
4. Viết W13, W14, sửa guard nếu cần, chạy `run.mjs workflows`.
5. Viết giao diện, `setup/sepay.mjs` và tài liệu hướng dẫn.
6. Thêm mục 11 SePay vào `verify.mjs` (mục 5 dưới đây). Dev phải "tất cả đạt", dùng khoá thử đặt trong `.env` của dev.
7. **Production:** sao lưu, cài plugin, chạy `run.mjs collections roles workflows ui`.
   - **Chưa** đặt `DSC_SEPAY_API_KEY` thật. Endpoint lúc này trả 503, đây là hành vi đúng.
   - Để chạy verify, đặt tạm một khoá thử ngẫu nhiên, chạy kịch bản với mã gói `[TEST]`, dọn dữ liệu, rồi **gỡ khoá thử**. Cách khác là chỉ verify phần 503/401 trên production và để phần còn lại cho dev. Tự chọn cách an toàn hơn và báo lại.
   - Chạy `parent-isolation.mjs`.
8. Commit Conventional Commits (tiếng Anh, có `Co-Authored-By`), cập nhật README, file bàn giao, mục 6e "Kết quả bước 2.5" của kế hoạch giai đoạn 2 và mục Tiến độ của kế hoạch chiến lược.

## 5. Kiểm chứng (`verify.mjs` mục 11)

1. Không có khoá hoặc sai khoá: 401. Thiếu biến môi trường: 503. Cả hai trường hợp không tạo bản ghi nào.
2. Payload hợp lệ có `DSC DK<mã gói thử>`: tạo đúng **1** phiếu thu (vietqr, hoc_phi, đúng số tiền và gói). `da_thu` và `cong_no` của gói cập nhật đúng (F3). Có 1 dòng `da_thu` trong `dsc_zalo_hang_doi` (W10).
3. Gửi lại cùng `id`: trả 200 trùng, không có phiếu thu thứ hai. Gửi 2 request song song cùng `id` cũng chỉ ra 1 phiếu thu.
4. Nội dung `DSCDK<mã>` không dấu cách vẫn ghép được.
5. Không có mã, mã không tồn tại, hoặc gói đã huỷ: thành `cho_xu_ly`, không có phiếu thu, quản lý nhận được thông báo (W13).
6. Tiền ra, hoặc tiền vào tài khoản khác: thành `bo_qua`.
7. Thanh toán thiếu: `mot_phan`. Thanh toán thừa: `thua`, và `ly_do` ghi chênh lệch.
8. Ghép tay: chọn gói rồi bấm W14 thì tạo 1 phiếu thu. Bấm lần hai không tạo thêm.
9. Cách ly: phụ huynh thử đọc `dsc_giao_dich_ngan_hang` thì bị 403 hoặc nhận rỗng. Giáo viên không đọc được.
10. Unit test plugin: tách mã, chuẩn hoá payload, so khoá bằng timing-safe, phân loại trạng thái.

## 6. Việc của chủ dự án (không chặn bước 1–8)

- Đăng ký SePay, liên kết VPBank 168686838, cấu hình webhook theo `docs/huong-dan-sepay.md`.
- Gửi khoá API qua kênh an toàn để đặt vào Dokploy. Không dán vào chat.
- Sau khi bật: chuyển thử một khoản nhỏ cho gói của chính chủ dự án, rồi xem phiếu thu và tin Zalo.

## 7. Rủi ro

- **Webhook giả mạo:** xác thực bằng khoá với so sánh timing-safe, chỉ chạy qua HTTPS, và chỉ nhận đúng số tài khoản. Có thêm `du_lieu_goc` để tra soát.
- **Ghi trùng tiền:** dùng khoá unique `ma_giao_dich_cong` và transaction.
- **Phụ huynh ghi sai nội dung chuyển khoản:** giao dịch rơi vào `cho_xu_ly`, có thông báo, và ghép tay bằng W14.
- **SePay đổi định dạng:** giữ `du_lieu_goc` và trả `cho_xu_ly` thay vì lỗi để không mất giao dịch.
- **Dữ liệu tài chính:** không dùng giao dịch thật để thử. Verify chỉ dùng gói `[TEST]` và dọn sạch sau khi chạy.
