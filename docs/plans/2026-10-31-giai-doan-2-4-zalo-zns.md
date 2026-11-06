# Kế hoạch chi tiết bước 2.4 — Thông báo Zalo OA / ZNS cho phụ huynh

Thuộc kế hoạch giai đoạn 2 `docs/plans/2026-10-31-giai-doan-2-cong-phu-huynh-zalo-vietqr.md` (mục 3.2, bước 2.4). Lập ngày 31/10/2026.

## 1. Rà soát tiến độ (31/10/2026)

| Bước                                                 | Trạng thái                                   | Bằng chứng                      |
| ---------------------------------------------------- | -------------------------------------------- | ------------------------------- |
| Giai đoạn 1 + bộ máy tính toán F0–F7/FN/FM           | Xong                                         | commit `9ca15fd`, `0a65f2d`     |
| 2.1 Cổng phụ huynh + plugin `dsc-guard`              | Xong, 22 tài khoản thật                      | `1fb2bd4`, `6c474ac`            |
| 2.2 VietQR hiển thị                                  | Xong (chưa xem tận mắt trên UI)              | `5aad99b`                       |
| 2.3 Tuyển sinh (form công khai, W11, W12, dashboard) | Xong, verify 123/0                           | `25ded42` (mục 6c kế hoạch GĐ2) |
| **2.4 Zalo ZNS**                                     | **Chưa bắt đầu**: trong repo chưa có mã Zalo | ← kế hoạch này                  |
| 2.5 Webhook SePay                                    | Chưa bắt đầu                                 | sau 2.4                         |

Lưu ý: mục "Tiến độ" của kế hoạch chiến lược vẫn ghi 2.3 là chưa làm. Cần sửa lại khi xong 2.4.

## 2. Mục tiêu và thay đổi so với thiết kế gốc

Phụ huynh nhận ZNS tự động trong 3 trường hợp: **(a) có nhận xét buổi học**, **(b) gói sắp hết buổi**, **(c) đã nhận học phí**. Tất cả dùng **chế độ thử** (chỉ gửi đến SĐT của chủ dự án) cho tới khi chủ dự án duyệt.

Kiểm tra lại mã nguồn thì thấy thiết kế gốc ở mục 3.2 có hai điểm phải sửa:

1. **Bản mã nguồn mở 2.2.36 không có `plugin-environment-variables`**, nên không có chỗ an toàn để giữ app secret khi dùng node Request. Nếu để secret và token trong cấu hình node Request hay trong collection thì quản trị nào cũng đọc được, và token còn bị ghi vào kết quả job.
2. Gọi API Zalo thẳng từ nhiều workflow thì logic "đồng ý dữ liệu / chế độ thử / chống trùng / ghi nhật ký" bị lặp ở từng nơi, dễ sót.

**Thiết kế đã chốt cho bước này: dùng hàng đợi gửi đi (outbox) cùng một plugin chỉ có phần server.**

- Workflow chỉ quyết định _khi nào_ gửi. Workflow chỉ **ghi dòng vào hàng đợi** `dsc_zalo_hang_doi` (bằng node SQL hoặc node create, không cần biết gì về Zalo).
- Plugin `@duongsinh/plugin-dsc-zalo` quyết định _có gửi không và gửi thế nào_. Mỗi 1 phút, plugin lấy các dòng `cho_gui`, kiểm tra đồng ý dữ liệu, chế độ thử và hạn mức, làm mới token, gọi ZNS, rồi ghi kết quả vào chính dòng đó. Bảng hàng đợi cũng là **nhật ký gửi** (thay `dsc_nhat_ky_zalo`).
- App ID và secret đặt trong **biến môi trường container** (Dokploy), không ghi vào DB. Token được giữ trong bảng kỹ thuật mà không vai trò nào (trừ root) có quyền đọc.
- W8 bị bỏ, vì plugin tự làm mới token khi token sắp hết hạn.

## 3. Thiết kế chi tiết

### 3.1 Collection mới (`setup/schema/phase1.mjs`, hoặc tách ra `setup/schema/zalo.mjs` rồi gọi từ `run.mjs collections`)

**`dsc_zalo_hang_doi`** (hàng đợi và nhật ký):

- `loai` (select: `nhan_xet`, `sap_het`, `da_thu`, `thu_nghiem`)
- `khoa_chong_trung` (string, **unique**, vd `nhan_xet:<diem_danh_id>:<phu_huynh_id>`)
- `so_dien_thoai` (string; lúc gửi đổi sang dạng `84…`)
- `du_lieu` (json, tham số mẫu)
- `trang_thai` (select: `cho_gui`, `da_gui`, `loi`, `bo_qua`), `so_lan_thu` (int), `loi` (text), `msg_id` (string), `gui_luc` (datetime)
- quan hệ m2o: `phu_huynh`, `hoc_vien`
- `createdAt`/`updatedAt` có sẵn.

**`dsc_zalo_cau_hinh`** (bảng một dòng, chỉ root đọc và sửa):

- `che_do` (select `tat` / `thu` / `that`, **mặc định `tat`**)
- `so_thu` (SĐT chủ dự án), `ma_mau_nhan_xet`, `ma_mau_sap_het`, `ma_mau_da_thu` (template_id)
- `han_muc_ngay` (int, mặc định 200)
- `access_token`, `refresh_token`, `token_het_han` (datetime). Ba trường này không bao giờ trả về qua API cho vai trò khác root.

ACL (`setup/acl/apply.mjs`):

- `dsc_quan_ly` chỉ được **xem** `dsc_zalo_hang_doi` để tra soát. Không cấp vai trò nào quyền với `dsc_zalo_cau_hinh`.
- Phải chắc chắn vai trò `dsc_phu_huynh` không có quyền gì với cả hai bảng. Thêm cả hai bảng vào bộ dò của `setup/parent-isolation.mjs`.

### 3.2 Plugin `@duongsinh/plugin-dsc-zalo` (`D:\code\duongsinh-nocobase\plugins\plugin-dsc-zalo\`)

Chép khung từ `plugins/plugin-dsc-guard/` (package.json, `src/index.ts`, `src/server/plugin.ts`, `server.d.ts`, `client.d.ts`). Build bằng `node plugins/build.mjs plugin-dsc-zalo`, cài bằng `plugins/install.mjs`.

- `src/server/zalo-api.ts`: các hàm thuần, có thể test riêng.
  - `refreshToken({ appId, secret, refreshToken })`: `POST https://oauth.zaloapp.com/v4/oa/access_token`, header `secret_key`, body form-urlencoded `refresh_token`, `app_id`, `grant_type=refresh_token`. Kết quả trả về là access token (hạn khoảng 25 giờ) và **refresh token mới**. Refresh token chỉ dùng được một lần, nên phải lưu token mới ngay trong cùng transaction.
  - `sendZns({ accessToken, phone, templateId, data, trackingId })`: `POST https://business.openapi.zalo.me/message/template`, header `access_token`, JSON `{ phone, template_id, template_data, tracking_id }`. Gửi thành công khi `error === 0`. Lấy `data.msg_id`.
  - `normalizePhone('0912…') → '84912…'`. Số không hợp lệ thì trả `null`.
  - **Trước khi viết, đối chiếu tài liệu chính thức Zalo** (developers.zalo.me → ZNS, OA API v3/v4) về tên trường, mã lỗi và giới hạn độ dài tham số. Nếu khác những gì ghi ở đây thì làm theo tài liệu.
- `src/server/worker.ts`: `processQueue(db)` chạy mỗi 60 giây bằng `setInterval`, đăng ký trong `afterStart` và dừng trong `beforeStop`. Có cờ chống chạy chồng.
  1. Đọc cấu hình. Nếu `che_do = 'tat'` thì dừng.
  2. Lấy tối đa 50 dòng `cho_gui` và `so_lan_thu < 3`, sắp theo `id`.
  3. Với từng dòng:
     - Phụ huynh không còn `dong_y_du_lieu` hoặc không có SĐT hợp lệ → `bo_qua`.
     - Đã vượt `han_muc_ngay` (đếm `da_gui` trong ngày) → dừng lượt này.
     - Đang ở chế độ `thu` thì thay SĐT người nhận bằng `so_thu`, và ghi SĐT gốc vào `du_lieu._goc`.
     - Gửi tin. Thành công → `da_gui`, `msg_id`, `gui_luc`. Lỗi → `so_lan_thu + 1` và lưu `loi`; tới lần thứ 3 thì đặt `loi`.
  4. Token sắp hết hạn (còn dưới 1 giờ) thì làm mới trước khi gửi. Làm mới thất bại thì gửi **thông báo trong app** cho quản lý qua kênh `dsc_in_app` (dùng API notification-manager của server), mỗi ngày tối đa một lần.
- Biến môi trường (đặt trong Dokploy compose `nocobase-dsc-jyvcqc` và trong `.env` của bản dev): `DSC_ZALO_APP_ID`, `DSC_ZALO_APP_SECRET`. Thiếu một trong hai biến thì plugin ghi log cảnh báo và không gửi gì.
- Action `dscZalo:setRefreshToken` (chỉ root) nhận refresh token ban đầu. Chủ dự án lấy token này một lần bằng công cụ API Explorer của Zalo cho OA, sau đó chạy `setup/zalo.mjs set-token`. Script đọc token từ stdin hoặc từ file `.secret`, **không in token ra**.
- Action `dscZalo:test` (chỉ root) đưa một dòng loại `thu_nghiem` vào hàng đợi, gửi tới `so_thu` để thử từ đầu đến cuối.
- Unit test (vitest hoặc `node:test`, theo cách repo đang test `setup/tests/*.test.mjs`) cho `normalizePhone`, cách dựng body, xử lý mã lỗi, và quy tắc chế độ thử. Giả lập `fetch`, **không gọi Zalo thật**.

### 3.3 Workflow ghi vào hàng đợi (`setup/workflows/apply.mjs` + module SQL mới `setup/workflows/zalo.mjs`)

Cả ba workflow dùng SQL `INSERT … SELECT … WHERE NOT EXISTS (khóa chống trùng)`, viết theo cú pháp chung của PostgreSQL và SQLite giống `formulas.mjs`, truyền tham số an toàn qua `:id`/`:hv`. Bảng nối phụ huynh–học viên là `dsc_hoc_vien_phu_huynh`. **Kiểm tra tên cột thật** (dự kiến `dscHocVienId`, `dscPhuHuynhId`, giống `dsc_lop_hoc_hoc_vien`). Điều kiện chung: `ph.dong_y_du_lieu = true AND ph.dien_thoai IS NOT NULL`, và học viên không có `lark_id` thì không cần lọc thêm.

| Workflow                                  | Kích hoạt                                                                                                                               | Ghi vào hàng đợi                                                                                                                                                                                                                                                                                                                                                                  |
| ----------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **W9** `dsc_w9_zns_nhan_xet`              | Lịch **20:30 hằng ngày** (gom tin để giáo viên kịp sửa nhận xét và để giảm phí)                                                         | Mỗi điểm danh của **buổi trong ngày hôm nay** có `nhan_xet` không rỗng ghi một dòng cho mỗi phụ huynh. Khoá chống trùng là `nhan_xet:<diem_danh_id>:<ph_id>`. Tham số gồm tên con, ngày buổi học, tên lớp và một câu ngắn "Xem nhận xét trong cổng phụ huynh". **Không** đưa toàn văn nhận xét vào tin, vì ZNS giới hạn độ dài tham số và phụ huynh đã đọc được đầy đủ trên cổng. |
| **F1/FN/FM mở rộng** (`notifyNearlyDone`) | Đã có sẵn                                                                                                                               | Thêm một node SQL ngay **trước** node `*_da_bao`, dùng cùng điều kiện `NOT_NOTIFIED` của `formulas.mjs` (`notifyStatements`), để ghi dòng `sap_het` cho mỗi phụ huynh. Khoá chống trùng là `sap_het:<goi_id>:<ph_id>`. Khi gói được gia hạn, cờ `thong_bao_sap_het_luc` bị xoá, nên lần sau khoá cần có thêm hậu tố ngày, vd `sap_het:<goi_id>:<ph_id>:<YYYY-MM-DD>`.             |
| **W10** `dsc_w10_zns_da_thu`              | Collection `dsc_phieu_thu`, mode 1 (tạo mới), điều kiện `trang_thai=da_thu`, `loai_thu=hoc_phi`, `lark_id` rỗng, `hocVienId` có giá trị | Khoá chống trùng là `da_thu:<phieu_id>:<ph_id>`. Tham số gồm tên con, số tiền, mã gói và ngày thu. Workflow này độc lập với F3 vì F3 chỉ tính lại số liệu.                                                                                                                                                                                                                        |

Ghi chú:

- Lần đầu bật thì đặt `che_do = 'tat'`. Hàng đợi vẫn được ghi để kiểm tra trước nội dung sẽ gửi, nhưng chưa gửi gì.
- Các gói đang "Sắp hết" đã được đánh dấu là đã báo từ trước, nên sẽ không có tin dồn.
- Workflow tạo bằng cơ chế dấu vân tay đã có, nên đổi định nghĩa thì sẽ sinh phiên bản mới.

### 3.4 Giao diện (`setup/ui/pages.mjs`)

- Thêm trang "Zalo ZNS" trong nhóm quản trị (chỉ `dsc_quan_ly`). Trang có một bảng `dsc_zalo_hang_doi` (mới nhất trước, lọc theo trạng thái và loại). Bảng **chỉ xem**, có thêm nút "Gửi lại" để đặt lại `trang_thai=cho_gui`, `so_lan_thu=0` cho dòng lỗi.
- Trang cấu hình chế độ chỉ cho root, vì có token. Cũng có thể bỏ trang này và đổi chế độ bằng `setup/zalo.mjs mode tat|thu|that`. **Khuyên dùng script** vì gọn và không lộ token.
- Nhớ khai báo `FIELD_GROUPS` khi bảng có hơn 10 trường (bẫy của validator).

### 3.5 Script `setup/zalo.mjs`

Các lệnh: `status` (chế độ, token còn hạn bao lâu, số dòng theo trạng thái), `mode <tat|thu|that> [--so-thu 09…]`, `templates --nhan-xet <id> --sap-het <id> --da-thu <id>`, `set-token` (đọc từ stdin), `test`. Script dùng `setup/lib/api.mjs`.

## 4. Thứ tự triển khai

1. **Đọc trước:** file bàn giao `2026-10-30-ban-giao-phien-tiep-theo.md` (bẫy Windows và NocoBase), README của `duongsinh-nocobase`, `setup/workflows/apply.mjs`, `formulas.mjs`, plugin `dsc-guard`. Chạy `git status`. Nếu cây làm việc chưa sạch thì dừng lại và báo.
2. Viết schema và ACL, chạy `run.mjs collections` và `run.mjs roles` trên dev.
3. Viết plugin, unit test, build, cài trên dev (junction `node_modules\@duongsinh\plugin-dsc-zalo`).
4. Viết `setup/workflows/zalo.mjs`, sửa `apply.mjs` (W9, W10, mở rộng `notifyNearlyDone`), chạy `run.mjs workflows`.
5. Viết giao diện và `setup/zalo.mjs`.
6. Mở rộng `verify.mjs` (mục 5), dev phải "tất cả đạt" với `che_do=tat` và Zalo giả lập.
7. **Production:** sao lưu (`ssh root@217.15.160.118 /usr/local/bin/nocobase-dsc-backup.sh`), cài plugin, `run.mjs collections roles workflows ui`, `verify.mjs` → "tất cả đạt", chạy lại `parent-isolation.mjs`. **Chưa** đặt biến môi trường, **chưa** bật `thu`.
8. Commit Conventional Commits (tiếng Anh, có `Co-Authored-By`), cập nhật README, file bàn giao, mục 6d "Kết quả bước 2.4" trong kế hoạch GĐ2 và mục Tiến độ của kế hoạch chiến lược (sửa luôn trạng thái 2.3 thành xong).
9. **Chờ chủ dự án** (xem mục 6), rồi bật `thu`, gửi thử tới số chủ dự án, chủ dự án duyệt, bật `that`.

## 5. Kiểm chứng (thêm vào `setup/verify.mjs`, mục "10 Zalo")

1. Điểm danh có nhận xét trong ngày, chạy W9 (gọi trực tiếp node hoặc chạy thủ công): mỗi phụ huynh _đồng ý_ có đúng một dòng `cho_gui`. Phụ huynh _không đồng ý_ không có dòng nào. Chạy W9 lần hai không có thêm dòng.
2. Gói xuống "Sắp hết": có một dòng `sap_het` cho mỗi phụ huynh. Chạy FM lại không tạo thêm. Gia hạn rồi lại sắp hết vào ngày khác thì có dòng mới.
3. Phiếu thu học phí mới "Đã thu": có một dòng `da_thu`. Phiếu tiền sách hoặc phiếu nhập từ Lark không tạo dòng nào.
4. `che_do=tat`: worker không đổi trạng thái dòng nào.
5. Unit test plugin: chế độ thử luôn thay người nhận bằng `so_thu`. Phụ huynh rút đồng ý thì dòng thành `bo_qua`. Lỗi mạng làm tăng `so_lan_thu`, tới 3 lần thì thành `loi`. Token mới được lưu sau khi làm mới. Vượt hạn mức thì dừng.
6. Cách ly: tài khoản phụ huynh thử đọc `dsc_zalo_hang_doi` và `dsc_zalo_cau_hinh` thì bị 403 hoặc nhận về rỗng. Quản lý không đọc được `dsc_zalo_cau_hinh`.
7. Dọn dữ liệu thử `[TEST]` sau khi kiểm tra trên production.

## 6. Việc của chủ dự án (không chặn bước 1–8)

- Tạo hoặc kiểm tra ứng dụng trên Zalo for Developers gắn với OA đã xác thực, lấy **App ID và App secret**, rồi gửi qua kênh an toàn để đặt vào Dokploy. Không dán vào chat.
- Đăng ký ZNS, nạp tiền, tạo và duyệt **3 mẫu**: nhận xét buổi học, sắp hết buổi, đã nhận học phí. Gửi lại `template_id` cùng danh sách tham số của từng mẫu. Implementer soạn sẵn nội dung đề xuất trong `duongsinh-nocobase/docs/zns-mau-de-xuat.md`.
- Lấy refresh token ban đầu cho OA (implementer viết hướng dẫn từng bước).
- Cung cấp SĐT nhận tin thử.

## 7. Rủi ro

- **Refresh token chỉ dùng một lần**: nếu lưu token mới thất bại thì mất quyền truy cập và phải lấy lại bằng tay. Vì vậy phải ghi token ngay sau khi nhận và báo quản lý khi lỗi.
- **Phí ZNS**: có hạn mức ngày và gom nhận xét theo ngày. Chế độ mặc định là `tat`.
- **Gửi nhầm cho phụ huynh thật trong lúc thử**: chế độ `thu` được ép ở tầng plugin chứ không chỉ ở workflow, và có unit test.
- **Dữ liệu cá nhân**: chỉ gửi khi `dong_y_du_lieu = true`, kiểm tra cả lúc ghi hàng đợi lẫn lúc gửi.
- **Tải VPS**: worker mỗi phút chỉ đọc tối đa 50 dòng nên rất nhẹ.
