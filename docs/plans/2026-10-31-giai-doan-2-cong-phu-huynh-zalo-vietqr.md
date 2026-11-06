# Kế hoạch giai đoạn 2 — Cổng phụ huynh, Zalo OA, thanh toán VietQR

Thuộc kế hoạch tổng `docs/plans/2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md` (giai đoạn 2: "Tuyển sinh & cổng phụ huynh"). Lập ngày 31/10/2026. Prompt triển khai cho phiên khác: `docs/plans/2026-10-31-prompt-trien-khai-giai-doan-2.md`.

## 1. Bối cảnh

Giai đoạn 1 đã chạy production tại https://nocobase.dsc.edu.vn (NocoBase 2.2.36 + PostgreSQL, Dokploy, sao lưu hằng đêm lên Google Drive): 13 collection `dsc_*`, workflow W0–W7, giao diện desktop + mobile giáo viên (`/v/mobile`), dữ liệu thật nhập từ Lark (175 học viên, 349 gói, 446 phiếu thu, ~5.600 lượt điểm danh có nhận xét, 37 sách, 338 lần phát sách). Hiện phụ huynh chỉ nhận thông tin qua Zalo cá nhân của giáo viên; tiền học thu bằng chuyển khoản/tiền mặt và ghi tay.

**Mục tiêu giai đoạn 2:** phụ huynh tự xem tiến bộ của con và học phí; trung tâm gửi thông báo tự động qua Zalo OA; phụ huynh thanh toán bằng mã VietQR và hệ thống tự ghi nhận; tuyển sinh có quy trình rõ ràng.

**Thước đo (từ kế hoạch tổng):** ≥ 70% phụ huynh đang có con học kích hoạt cổng; thời gian xử lý lead < 24h; ≥ 80% phiếu thu học phí mới được ghi nhận tự động.

## 2. Hiện trạng kỹ thuật liên quan (đã kiểm chứng trong mã nguồn 2.2.36)

| Năng lực                  | Có sẵn                                                                      | Ghi chú                                                                                                                |
| ------------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| Đăng nhập SMS OTP         | `plugin-auth-sms` + `plugin-verification`                                   | Nhà cung cấp SMS chỉ có **Aliyun, Tencent** → cần provider Việt Nam tự viết, hoặc bắt đầu bằng mật khẩu                |
| Thông báo trong app       | `plugin-notification-in-app-message`                                        | Đang dùng (kênh `dsc_in_app`)                                                                                          |
| Gọi API ngoài từ workflow | `plugin-workflow-request`                                                   | Đủ để gọi Zalo ZNS API, không cần plugin                                                                               |
| Trigger workflow          | `collection`, `schedule`, `action`, `custom-action`, `request-interception` | **Không có webhook** trong bản mã nguồn mở → nhận webhook ngân hàng cần plugin nhỏ                                     |
| Trang mobile / nhiều cổng | `flowSurfaces` layout `mobile-layout-model`                                 | `listNavigationTargets` báo `multiPortal: false` → phụ huynh dùng chung `/v/mobile`, tách bằng quyền menu theo vai trò |
| Biểu mẫu công khai        | `plugin-public-forms`                                                       | Dùng cho đăng ký học thử                                                                                               |

## 3. Thiết kế

### 3.1 Cổng phụ huynh (mobile, `/v/mobile`)

- **Tài khoản:** `dsc_phu_huynh` thêm quan hệ `tai_khoan` (m2o → `users`). Script `setup/parents.mjs` tạo tài khoản cho phụ huynh có SĐT: `username` = SĐT chuẩn hoá, mật khẩu tạm ngẫu nhiên (ghi file `.secret`), vai trò mới `dsc_phu_huynh` (vai trò duy nhất của họ). Bước 2 (khi có nhà cung cấp SMS/ZNS): đăng nhập bằng OTP.
- **Vai trò `dsc_phu_huynh` — chỉ xem dữ liệu con mình (data scope theo `{{$user.id}}`):**
  - `dsc_hoc_vien`: `phu_huynh.tai_khoan.id = $user.id`
  - `dsc_diem_danh`, `dsc_dang_ky_goi`, `dsc_phieu_thu`, `dsc_giao_dich_sao`, `dsc_phat_sach`: `hoc_vien.phu_huynh.tai_khoan.id = $user.id`
  - `dsc_lop_hoc`, `dsc_sach`, `dsc_co_so`, `dsc_cap_do`, `dsc_goi_hoc`: chỉ xem; không có quyền ghi gì.
  - Trường nhạy cảm của trung tâm (ghi chú nội bộ, `lark_id`, `ma_cu`) không hiển thị trên trang phụ huynh (khai báo cột tường minh).
  - Kiểm thử bắt buộc: phụ huynh A **không** đọc được học viên / phiếu thu của phụ huynh B (HTTP 200 nhưng rỗng, hoặc 403).
- **Trang mobile cho phụ huynh** (quyền menu chỉ cấp cho `dsc_phu_huynh`; giáo viên không thấy và ngược lại):
  1. _Con tôi_: hồ sơ, lớp, cấp độ, sách đang học, tổng sao.
  2. _Nhận xét_: danh sách buổi học + nhận xét gửi phụ huynh (mới nhất trước), hiện ngày bằng `buoi_hoc.ngay` (bẫy đã biết).
  3. _Học phí_: các gói (số buổi, đã học, còn lại, trạng thái), phiếu thu; nút **"Thanh toán VietQR"** khi gói "Chờ thanh toán"/"Sắp hết".
- **Kích hoạt:** in / gửi hướng dẫn đăng nhập (link + tài khoản tạm) qua Zalo; đếm tỷ lệ kích hoạt bằng `users.lastSignInAt` (hoặc nhật ký đăng nhập).

### 3.2 Zalo OA / ZNS

- **Điều kiện (chủ dự án lo):** Zalo Official Account đã xác thực; đăng ký ZNS; ứng dụng Zalo Developer (App ID, secret key); duyệt các **mẫu ZNS**: (1) nhận xét buổi học, (2) sắp hết buổi / gia hạn, (3) xác nhận đã nhận học phí, (4) OTP đăng nhập (nếu dùng).
- **Lưu token:** collection `dsc_cau_hinh` (khoá–giá trị, chỉ `dsc_quan_ly`/root đọc) giữ `zalo_access_token`, `zalo_refresh_token`, `zalo_token_het_han`. Workflow lịch **W8** (mỗi 20 giờ) gọi `https://oauth.zaloapp.com/v4/oa/access_token` bằng refresh token (node Request) → cập nhật bảng cấu hình. App secret lưu ở **biến môi trường NocoBase** (`plugin-environment-variables`), không ghi vào collection.
- **Gửi ZNS bằng node Request** (`https://business.openapi.zalo.me/message/template`, header `access_token`), số điện thoại dạng `84xxxxxxxxx`:
  - **W9** nhận xét buổi học: khi điểm danh có `nhan_xet` (tạo/sửa) → gửi cho phụ huynh của học viên (có thể gom 1 tin/buổi/phụ huynh). Chỉ gửi khi phụ huynh có SĐT và `dong_y_du_lieu = true`.
  - **W1 mở rộng**: nhánh "Còn ≤ 2 buổi" gửi thêm ZNS gia hạn cho phụ huynh.
  - **W4 mở rộng / W10**: phiếu thu mới "Đã thu" → ZNS xác nhận.
  - Ghi nhật ký gửi vào `dsc_nhat_ky_zalo` (thời điểm, phụ huynh, mẫu, kết quả) để tra soát và tránh gửi trùng.
- **Chế độ thử:** cờ cấu hình `zalo_che_do_thu = true` → chỉ gửi đến SĐT của chủ dự án; bật thật khi đã kiểm tra mẫu.

### 3.3 Thanh toán VietQR + tự ghi nhận

- **Hiển thị QR (không cần khoá API):** ảnh `https://img.vietqr.io/image/<BANK_BIN>-<SO_TK>-compact2.png?amount=<so_tien>&addInfo=<NOI_DUNG>&accountName=<TEN_TK>`. Nội dung chuyển khoản chuẩn: `DSC <ma_dang_ky>` (vd `DSC DK00352`) để đối soát. Hiển thị bằng khối markdown/JS trên trang Học phí của phụ huynh và trong popup gói (phía trung tâm).
- **Tự ghi nhận (chọn 1 cổng, chủ dự án đăng ký):** SePay hoặc Casso đọc biến động tài khoản ngân hàng và gọi webhook.
  - Plugin nhỏ **`@duongsinh/plugin-dsc-bank-webhook`** (repo riêng, đặt ở `packages/pro-plugins/@duongsinh/` hoặc build thành `.tgz` cài qua Plugin Manager/Dokploy): endpoint `POST /api/dscBankWebhook:receive`, xác thực bằng API key/HMAC của cổng, chống trùng theo mã giao dịch, tách `DK\d{5}` trong nội dung → tạo `dsc_phieu_thu` (loại "Học phí", hình thức "VietQR"/"Chuyển khoản", gắn gói + học viên) → W4 tự cộng "đã thu". Giao dịch không nhận diện được → ghi vào `dsc_giao_dich_ngan_hang` (chờ xử lý tay) + thông báo cho quản lý.
  - Kiểm thử bằng payload mẫu của cổng; không dùng thẻ / tài khoản thật để thử.

### 3.4 Tuyển sinh (CRM lead)

- Biểu mẫu công khai "Đăng ký học thử" (`plugin-public-forms`) → tạo `dsc_lead` (nguồn = website/facebook theo tham số).
- **W11**: lead mới → thông báo trong app cho vai trò `dsc_tu_van` + quản lý; lead "Mới" quá 24 giờ chưa đổi trạng thái → nhắc lại (schedule).
- Kanban lead đã có; thêm dashboard tỷ lệ chuyển đổi theo nguồn (SQL tương thích PostgreSQL).
- Chuyển lead "Đã đăng ký" → tạo học viên + phụ huynh (nút cập nhật hoặc workflow thủ công).

## 4. Lộ trình

| Bước | Nội dung                                                                                                                                         | Phụ thuộc chủ dự án                      |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------- |
| 2.0  | Chốt quyết định (mục 6), chuẩn bị tài khoản Zalo OA / ngân hàng / cổng webhook                                                                   | Có                                       |
| 2.1  | Cổng phụ huynh: `tai_khoan` trên phụ huynh, vai trò `dsc_phu_huynh` + data scope, 3 trang mobile, script tạo tài khoản, kiểm thử cách ly dữ liệu | Danh sách SĐT phụ huynh (đã có một phần) |
| 2.2  | VietQR hiển thị (không cần cổng)                                                                                                                 | Số TK, ngân hàng, tên chủ TK             |
| 2.3  | Tuyển sinh: form công khai, W11, dashboard                                                                                                       | —                                        |
| 2.4  | Zalo OA: bảng cấu hình, W8 làm mới token, W9/W1/W10 ở chế độ thử → bật thật                                                                      | OA đã xác thực, mẫu ZNS được duyệt       |
| 2.5  | Webhook ngân hàng: plugin + chống trùng + đối soát                                                                                               | Tài khoản SePay/Casso + khoá API         |
| 2.6  | Kích hoạt phụ huynh hàng loạt, đo thước đo, tinh chỉnh                                                                                           | —                                        |

Mỗi bước: thử ở dev (SQLite) → production → `verify.mjs` mở rộng → sao lưu → commit trong `D:\code\duongsinh-nocobase` → cập nhật mục Tiến độ của kế hoạch tổng.

## 5. Rủi ro

- **Rò rỉ dữ liệu giữa các gia đình** (nghiêm trọng nhất): data scope sai → phụ huynh xem được con nhà khác. Bắt buộc kiểm thử tự động 2 phụ huynh × 2 học viên trên mọi collection trước khi cấp tài khoản thật.
- Dữ liệu phụ huynh còn thiếu (92 học viên đang theo học chưa gắn phụ huynh) → cần bổ sung trước khi kích hoạt hàng loạt.
- Token Zalo hết hạn → W8 lỗi thì ZNS dừng: thông báo cho quản lý khi làm mới token thất bại.
- ZNS tính phí theo tin: gom tin, chỉ gửi khi có nhận xét; đặt hạn mức.
- Webhook giả mạo → bắt buộc xác thực chữ ký/khoá, chỉ nhận HTTPS, chống trùng mã giao dịch.
- Luật Bảo vệ dữ liệu cá nhân: chỉ gửi Zalo/cấp cổng khi `dong_y_du_lieu = true`; thu thập đồng ý ở bước kích hoạt.
- Bẫy NocoBase đã gặp (xem file bàn giao): popup 2 bảng liên quan, quan hệ có tên hiển thị là ngày, decimal về dạng chuỗi trong workflow.

## 6. Quyết định cần chủ dự án chốt khi bắt đầu

1. Phụ huynh đăng nhập bằng **mật khẩu tạm** trước (khuyên dùng), hay chờ **OTP** (cần nhà cung cấp SMS Việt Nam hoặc ZNS OTP)?
2. Tài khoản ngân hàng nhận học phí (ngân hàng, số TK, tên chủ TK) và cổng tự ghi nhận (**SePay** hay **Casso**).
3. Zalo OA đã xác thực chưa; ai duyệt nội dung 3–4 mẫu ZNS.
4. Nội dung phụ huynh được xem: nhận xét đầy đủ hay chỉ phần "Nhận xét gửi phụ huynh"; có hiện phiếu thu chi tiết không.
5. Ưu tiên thứ tự 2.1–2.5 (khuyên: 2.1 → 2.2 → 2.3 → 2.4 → 2.5).

**Đã chốt (31/10/2026):**

1. Mật khẩu tạm trước; OTP làm sau khi có ZNS.
2. Cổng tự ghi nhận: **SePay**. Tài khoản nhận học phí: **VPBank (BIN 970432), số 168686838, tên chủ TK "DUONG SINH CHESS JSC"** (Công ty CP Cờ vua Dương Sinh, MST 0108526687).
3. Zalo OA **đã xác thực**; chủ dự án tự duyệt nội dung mẫu ZNS.
4. Phụ huynh xem **chỉ "Nhận xét gửi phụ huynh"** (+ điểm chấm), ẩn ghi chú nội bộ; **có** hiện phiếu thu chi tiết.
5. Thứ tự: 2.1 → 2.2 → 2.3 → 2.4 → 2.5.

## 6b. Kết quả bước 2.1 (31/10/2026)

- Đúng như rủi ro mục 5: chỉ cấu hình data scope thì **chưa đủ**. NocoBase 2.2.36 lọc bản ghi chính nhưng không lọc dữ liệu nạp kèm (`appends=lop_hoc.hoc_vien.phieu_thu`…) và route quan hệ; whitelist appends trong quyền cũng không có tác dụng. Bộ kiểm thử bắt được 146 đường rò rỉ ở lần chạy đầu.
- Khắc phục: plugin `@duongsinh/plugin-dsc-guard` (chặn trước ACL cho tài khoản chỉ có vai trò phụ huynh), dùng lại cấu hình ACL của vai trò làm danh sách cho phép. Sau khi bật: 0 rò rỉ.
- Trang mobile phụ huynh đặt tên "Gói học phí" (không phải "Học phí") để không trùng nhóm desktop.
- Plugin này cũng là khung cho plugin webhook SePay ở bước 2.5 (build bằng `plugins/build.mjs`, cài bằng `plugins/install.mjs`).

## 6c. Kết quả bước 2.3 Tuyển sinh (31/10/2026)

- Biểu mẫu công khai "Đăng ký học thử": https://nocobase.dsc.edu.vn/v/public-forms/a13df567c56 (plugin `public-forms` đã bật trên production).
- Thay đổi so với thiết kế mục 3.4: form **không** ghi thẳng `dsc_lead`. `publicSubmit` bỏ qua ACL nên người ngoài gửi được cả trường quan hệ, vì vậy form ghi vào bảng trung gian `dsc_dang_ky_web` (chỉ trường thường). W11 chép sang lead, và chỉ chép khi phụ huynh tick đồng ý xử lý dữ liệu.
- W11: báo lead mới; 08:30 nhắc lead "Mới" quá 24 giờ (một lần). W12: nút "Chuyển thành học viên" trên lead. Dashboard: tỷ lệ đăng ký theo nguồn.
- `verify.mjs` production: 123 đạt / 0 trượt (mục 9 tuyển sinh, gồm 7 phép dò). Commit `25ded42`.
- Còn mở: chống spam form (chưa có captcha/giới hạn tần suất), sẽ theo dõi số lượng; cần thì thêm.

## 6d. Kết quả bước 2.4 Zalo ZNS (31/10/2026)

- Kế hoạch chi tiết: `2026-10-31-giai-doan-2-4-zalo-zns.md`. Commit `f69b6d5` (duongsinh-nocobase). Chế độ production: `tat`; chưa đặt biến môi trường, chưa có token, chưa gửi tin nào.
- Chống trùng đã kiểm: W9 chạy 2 lần và FM chạy lại không sinh thêm dòng (SELECT đã loại khoá có sẵn); khoá vẫn là cột unique ở DB và vòng lặp dùng `exit: 2` nên một dòng trùng/lỗi không làm hỏng workflow.
- Kiểm chứng: dev `verify.mjs` 147 đạt / 0 trượt; production 157 đạt / 0 trượt (gồm mục 10 Zalo và cách ly phụ huynh 296 phép dò); 16 unit test plugin với Zalo giả lập.
- Lệch so với kế hoạch: (1) id các bảng là snowflake, không có giá trị mặc định ở DB nên workflow **không INSERT thô được** → node SQL chỉ SELECT, vòng lặp + node create ghi từng dòng; (2) trang tài liệu chính thức Zalo là SPA không đọc được bằng công cụ, endpoint/tham số/mã lỗi dựa SDK bên thứ ba (`oauth.zaloapp.com/v4/oa/access_token`, header `secret_key`, `business.openapi.zalo.me/message/template`, `error === 0`, mã -124 = token hỏng) → **cần đối chiếu khi test thật**; (3) "Gửi lại" = popup sửa đúng 3 trường (ACL giới hạn trường); (4) thêm action root `dscZalo:runNow` để chạy tay một lượt; (5) `verify.mjs` chờ workflow nền yên trước mục 9–10 (SQLite dev bị SQLITE_BUSY).

### PHẢI xác minh trước khi bật `che_do=thu`

Các điểm sau lấy từ SDK bên thứ ba, chưa đối chiếu tài liệu chính thức Zalo (developers.zalo.me là SPA, công cụ không đọc được). Chủ dự án hoặc người triển khai mở tài liệu / API Explorer và kiểm từng mục; sai thì sửa ở `plugins/plugin-dsc-zalo/src/server/zalo.ts` (hằng số `OAUTH_URL`, `ZNS_URL`, `TOKEN_ERROR_CODES`, hàm `refreshToken`, `sendZns`):

1. URL làm mới token OA: `POST https://oauth.zaloapp.com/v4/oa/access_token`, body form-urlencoded `refresh_token`, `app_id`, `grant_type=refresh_token`.
2. Tên header chứa app secret khi làm mới token: `secret_key`.
3. Phản hồi làm mới: `access_token`, `refresh_token` mới, `expires_in` (giây); refresh token dùng một lần.
4. URL gửi ZNS: `POST https://business.openapi.zalo.me/message/template`, header `access_token`, JSON `phone` (dạng `84…`), `template_id`, `template_data`, `tracking_id` (≤ 48 ký tự).
5. Thành công khi `error === 0`, mã tin ở `data.msg_id`.
6. Mã lỗi token hết hạn/không hợp lệ: đang coi `-124` (và `216`) là token hỏng → làm mới một lần rồi gửi lại. Các mã lỗi khác (số không dùng Zalo, mẫu chưa duyệt, hết tiền ZNS…) chưa phân loại: đều thử tối đa 3 lần rồi thành `loi`.
7. Giới hạn độ dài / định dạng từng tham số của mẫu.

Cách kiểm an toàn: chế độ `thu` + `node setup/zalo.mjs test --mau nhan_xet|sap_het|da_thu` (gửi tới số của chủ dự án), đọc cột `loi` ở trang "Hàng đợi & nhật ký ZNS".

- Chờ chủ dự án: xem `duongsinh-nocobase/docs/zns-mau-de-xuat.md`.

## 6e. Kết quả bước 2.5 Webhook SePay (01/11/2026)

- Kế hoạch chi tiết: `2026-11-01-giai-doan-2-5-webhook-sepay.md`. Commit `b701122` (duongsinh-nocobase). Production: plugin đã cài, **chưa đặt `DSC_SEPAY_API_KEY`** → endpoint trả 503 và không ghi gì; sổ giao dịch trống.
- Kiểm chứng: dev `verify.mjs` 189 đạt / 0 trượt (có khoá thử trong `.env` dev, gồm mục 11 đầy đủ); production 173 đạt / 0 trượt (mục 11 chỉ phần 503 + ghép tay W14 + phân quyền; phần gọi webhook không chạy vì không đặt khoá thử lên production). 12 unit test plugin. Số lần 5xx trên PostgreSQL: 0 (W14); webhook trên PostgreSQL **chưa thử** (xem dưới).
- Lệch so với kế hoạch: (1) trả đúng `{"success": true}` cả khi trùng (SePay đòi body y nguyên); (2) `dsc-guard` không cần sửa: W14 đi qua `dsc_giao_dich_ngan_hang:trigger` (ACL), không qua `workflows:trigger`; verify thêm phép dò giáo viên / phụ huynh / member gọi trigger → 403; (3) W14 có bước "giữ chỗ" (đặt giao dịch sang `loi` trước khi tạo phiếu thu) để lỗi giữa chừng không dẫn tới bấm lại tạo phiếu thứ hai; (4) có thêm khối dashboard "Giao dịch ngân hàng chờ xử lý"; (5) test dev thử lại khi webhook/W14 gặp SQLITE_BUSY và ghi số lần 5xx.
- Chưa kiểm được trên PostgreSQL (cần khoá thử trên môi trường PG): rollback khi lỗi giữa chừng (ca `id` dài 300 ký tự vượt varchar(255) trong verify 11.10 — SQLite không lỗi nên bỏ qua), chống trùng hai request song song trên PG, số 5xx của webhook. Cách làm: đặt tạm một khoá thử ngẫu nhiên lên production (Dokploy), chạy `DSC_SEPAY_API_KEY=<khoá thử> node setup/verify.mjs sepay`, rồi gỡ khoá — cần chủ dự án cấp lại Dokploy API key / thao tác.

## 7. Kiểm chứng

- `setup/verify.mjs` thêm: cách ly dữ liệu 2 phụ huynh; phụ huynh không ghi được gì; QR có đúng số tiền và nội dung `DSC DK…`; webhook mẫu tạo đúng 1 phiếu thu, gửi lại cùng mã giao dịch không tạo thêm; ZNS ở chế độ thử chỉ gửi tới số thử.
- Kiểm tra giao diện trên điện thoại thật bằng tài khoản phụ huynh thử (không phải root).
