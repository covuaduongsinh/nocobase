# Kiểm kê hệ thống Dương Sinh trên NocoBase 2 (giai đoạn B) — 03/11/2026

> Đầu vào cho việc dựng lại trên NocoBase 3 (kế hoạch `../2026-11-03-chuyen-production-nocobase-v2-sang-v3.md`, giai đoạn B).
> Bản máy đọc được: `kiem-ke-v2.json` cùng thư mục (đủ từng trường, quyền, workflow, trang).
>
> **Nguồn và cách làm (chỉ đọc):** mã trên máy chủ dự án `D:\code\duongsinh-nocobase` @ `cff46e1` + thay đổi chưa commit
> (phần AI), đối chiếu GitHub `main` @ `0c2bb8c` (**chậm 11 commit**: giải đấu UI, bán hàng, AI chưa push). Dữ liệu khai báo
> lấy bằng cách import các module thuần `setup/schema|acl|workflows|ui` — không gọi API, không chạm production. Tình trạng
> "đã lên production" lấy từ các file kế hoạch giai đoạn 1–4 và README; **mức dùng thật chưa đo** (cần chủ dự án chạy câu SQL ở
> mục 9).

## 1. Tổng quan con số

| Hạng mục                     | Số lượng                                                                                                              |
| ---------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Collection nghiệp vụ `dsc_*` | **49** (517 trường + quan hệ, 8 bảng trung gian n-n, 13 mã tự tăng)                                                   |
| Vai trò                      | **6** (`dsc_quan_ly`, `dsc_giao_vien`, `dsc_ke_toan`, `dsc_tu_van`, `dsc_ban_hang`, `dsc_phu_huynh`) + `root`/`admin` |
| Workflow                     | **55** (phần lớn là node SQL tính lại trường tự tính)                                                                 |
| Trang                        | **41** desktop (11 nhóm menu) + **12** mobile (6 giáo viên, 5 phụ huynh, 1 bán sách)                                  |
| Trang/form công khai         | 3 (đăng ký học thử, đăng ký giải đấu, cửa hàng sách)                                                                  |
| Plugin `@duongsinh/*`        | **7** (guard, chess, tournament, shop, bank-webhook, zalo, ai) ≈ 7.900 dòng TS                                        |
| Kiểm thử                     | `verify.mjs` mục 1–16 (production đạt 846 phép, gồm cách ly phụ huynh) + 22 file unit test + CI PR #1                 |

Năng lực v3 dùng để đối chiếu (gói đã phát hành 03/11/2026): `app-server`, `db` (+ postgres), `authorization`
(permission set, data scope, restriction/sharing rules), `authentication`, `users`, `file`, `notification` (+ in-app,
providers, web-push), `mail`, `scheduler`, `template-print`, `i18n`, `workflow` 2.0.0-beta.1 (DSL bằng mã),
`ai-employee` 3.0.0-beta.1, UI Library (shadcn/React). **Chưa có** gói v3 cho: form công khai, mobile, biểu đồ/dashboard,
lịch/kanban, nhập–xuất Excel, đăng nhập SMS/OTP, nhật ký kiểm toán, sao lưu trong app.

## 2. Collection theo mảng

Ký hiệu công sức dựng lại: S (≤ 1 ngày), M (2–4 ngày), L (≥ 1 tuần) — tính cả schema, quyền, logic tự tính, giao diện.

| Mảng           | Collection                                                                                                                                                             | Tình trạng                                                       | Ánh xạ v3                                                                   | Công sức |
| -------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- | --------------------------------------------------------------------------- | -------- |
| Vận hành CLB   | `co_so`, `cap_do`, `nhan_su`, `phu_huynh`, `hoc_vien`, `lop_hoc`, `buoi_hoc`, `sach`, `phat_sach`, `goi_hoc`, `dang_ky_goi`, `diem_danh`, `phieu_thu`, `giao_dich_sao` | Đã lên production (GĐ1–2)                                        | Collection bằng mã; trường tự tính → service/hook trong transaction         | L        |
| Tuyển sinh     | `lead`, `dang_ky_web`                                                                                                                                                  | Production (form công khai)                                      | Collection + trang công khai **tự viết**                                    | S        |
| Thanh toán     | `giao_dich_ngan_hang`                                                                                                                                                  | Production (webhook SePay) — cần xác nhận đã nhận giao dịch thật | Route public tự viết, dùng lại `receive.ts`                                 | M        |
| Zalo ZNS       | `zalo_hang_doi`, `zalo_cau_hinh`                                                                                                                                       | **Đã dựng chưa dùng** (chế độ `tat`, chưa có token OA)           | Notification provider tự viết                                               | M        |
| Học liệu       | `bai_hoc`, `the_co`, `giao_bai`, `lam_bai`                                                                                                                             | Production (GĐ3.1–3.2)                                           | Collection + trang React bàn cờ; dùng lại `grading.ts`                      | L        |
| Thi cấp        | `de_thi`, `ky_thi`, `bai_thi`, `chung_nhan`, `lich_su_cap`                                                                                                             | Production (GĐ3.3)                                               | Như trên + `template-print` cho chứng nhận                                  | M        |
| Giải đấu       | `giai_dau`, `bang_dau`, `ky_thu`, `van_dau`, `elo_lich_su`, `giay_khen`                                                                                                | Production (GĐ4.1), commit chưa push                             | Collection + route; dùng lại `core/{swiss,roundRobin,standings,elo,trf}.ts` | L        |
| Bán hàng & kho | `san_pham`, `doi_tac`, `kho`, `ton_kho`, `lan_in`, `khach_hang`, `don_hang`, `don_hang_dong`, `doi_soat`, `doi_soat_dong`, `phieu_kho`, `phieu_kho_dong`               | Production (GĐ4.2, 03/11), commit chưa push                      | Collection + service; dùng lại `stock.ts`, parser Shopee                    | L        |
| AI             | `ai_cau_hinh`, `ai_nhat_ky`, `bao_cao_thang`                                                                                                                           | **Dở dang** (chỉ dev, provider giả lập)                          | `ai-employee` hoặc tự viết; giữ `privacy.ts`                                | M        |

Chi tiết từng trường (kiểu, lựa chọn, mặc định, mã tự tăng, quan hệ, bảng trung gian) nằm trong `kiem-ke-v2.json` →
`collections[]`. Điểm cần nhớ khi dựng lại:

- **Mã tự tăng** (`sequence`, 5 chữ số): `HV` học viên, `DK` đăng ký gói, `PT` phiếu thu, `BH` bài học, `TC` thế cờ, `DT` đề
  thi, `CN` chứng nhận, `GIAI` giải đấu, `GD` kỳ thủ, `GK` giấy khen, `DH` đơn hàng, `DS` đối soát, `PK` phiếu kho. Khi
  chuyển phải **giữ nguyên mã cũ** và đặt bộ đếm v3 tiếp sau giá trị lớn nhất.
- **Trường tự tính** (không cho nhập tay): `hoc_vien.{tong_sao, so_buoi_con_lai, so_bai_tap_tuan, ngay_hoc_cuoi,
sinh_nhat_toi, so_du_tien_sach, elo, so_van_elo}`, `dang_ky_goi.{so_buoi_da_hoc, so_buoi_con_lai, da_thu, cong_no,
tt_thanh_toan, don_gia_buoi, ngay_hoc_cuoi, trang_thai}`, `lop_hoc.si_so`, `giao_bai.{si_so, so_da_xong}`,
  `ky_thu.{diem, buchholz*, sonneborn, so_thang, hang, da_thu}`, `san_pham.{ton_kho_tong, ton_ky_gui}`, `ton_kho.so_luong`,
  `don_hang.{tong_hang, tong_tien, da_thu, con_no}`, `doi_tac.cong_no`, `doi_soat.tong_tien`. Nguồn công thức duy nhất:
  `setup/workflows/formulas.mjs` (SQL chung PostgreSQL/SQLite) — dịch sang service v3 và giữ `recalc.mjs` làm phép đối soát.
- `lark_id` / `ma_cu` trên nhiều bảng: khoá nhập từ Lark — giữ khi chuyển để nhập lại được.
- Bảng trung gian n-n: `dsc_hoc_vien_phu_huynh`, `dsc_hoc_vien_sach_dang_hoc`, `dsc_hoc_vien_sach_da_hoc`,
  `dsc_lop_hoc_hoc_vien`, `dsc_diem_danh_sach`, `dsc_giao_bai_the_co`, `dsc_de_thi_the_co`, `dsc_ky_thi_hoc_vien`.
- Collection hệ thống v2 cần mang dữ liệu: `users` (tài khoản nhân viên + phụ huynh), `roles`/`rolesUsers` (gán vai trò),
  `attachments` (tệp nếu có). Không mang: workflows/executions/jobs, uiSchemas/flowModels (cấu hình v2).

## 3. Vai trò và quyền (phải dựng lại và kiểm lại y nguyên)

Nguồn: `setup/acl/apply.mjs`, `setup/ui/menu-access.mjs`, plugin `dsc-guard`. Bản đủ từng action/cột/phạm vi: JSON →
`roles[]`, `menuAccess`.

| Vai trò                             | Phạm vi                                                                                                                                                                                                                                                                                                        |
| ----------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dsc_quan_ly` Quản lý               | Toàn quyền (xem/tạo/sửa/xoá/xuất/nhập Excel) trên 47 collection; hàng đợi Zalo và giao dịch ngân hàng chỉ sửa vài cột trạng thái; chạy được workflow "nút bấm"                                                                                                                                                 |
| `dsc_giao_vien` Giáo viên/trợ giảng | Xem danh mục; sửa học viên; tạo/sửa buổi học, điểm danh, giao dịch sao, phát sách, học liệu, đề/kỳ thi; bài thi chỉ sửa `thuc_hanh`, `nhan_xet`; giải đấu: tạo/sửa kỳ thủ (14 cột) và ván đấu (vai trọng tài); **kho/tồn kho chỉ của cơ sở mình** (`co_so.nhan_su.tai_khoan = $user`); nhật ký AI chỉ của mình |
| `dsc_ke_toan` Kế toán               | Xem + xuất mọi thứ vận hành; tạo/sửa gói học, đăng ký gói, phiếu thu, phát sách; giao dịch ngân hàng sửa 5 cột + chạy ghép tay; kỳ thủ chỉ sửa `le_phi`; đơn hàng sửa 4 cột hoá đơn; đối soát ký gửi                                                                                                           |
| `dsc_tu_van` Tư vấn                 | Xem danh mục; tạo/sửa phụ huynh, học viên; toàn quyền lead                                                                                                                                                                                                                                                     |
| `dsc_ban_hang` Bán hàng             | Đơn hàng/dòng đơn/khách hàng tạo-sửa theo danh sách cột; xem sản phẩm, kho, tồn, phiếu kho                                                                                                                                                                                                                     |
| `dsc_phu_huynh` Phụ huynh           | **Chỉ xem, chỉ dữ liệu của con mình**, danh sách cột duyệt sẵn cho từng collection (27 collection)                                                                                                                                                                                                             |

**Quy tắc cách ly phụ huynh (ngữ nghĩa chính xác):**

1. Gốc là tài khoản đăng nhập: `dsc_phu_huynh.tai_khoan = $user.id`. Mọi bảng con lọc theo đường dẫn tới gốc này, ví dụ học
   viên `phu_huynh.tai_khoan`, điểm danh `hoc_vien.phu_huynh.tai_khoan`, buổi học `diem_danh.hoc_vien.phu_huynh.tai_khoan`,
   ván đấu `trang.hoc_vien…` **hoặc** `den.hoc_vien…`.
2. Điều kiện thêm: bài học chỉ `trang_thai = xuat_ban` và đúng cấp độ của con; thế cờ chỉ đã xuất bản và **không**
   `chi_dung_thi`; lần làm bài chỉ bài tập (không thuộc bài thi); bài thi chỉ khi `dat`/`khong_dat`; giải đấu/bảng đấu bỏ
   bản nháp; báo cáo tháng chỉ `da_duyet`.
3. Danh mục (`lop_hoc`, `sach`, `co_so`, `cap_do`, `goi_hoc`) xem được nhưng chỉ các cột duyệt sẵn.
4. Phụ huynh **không có quyền ghi** nào; không có quyền gì ở mảng bán hàng.
5. `dsc-guard` (vì NocoBase 2 không áp phạm vi cho dữ liệu nạp kèm): với tài khoản chỉ có vai trò phụ huynh — cấm route quan hệ
   (`a/1/b:list`); collection `dsc_*` chưa có trong danh sách cột duyệt (`parent-columns.ts`) → **từ chối (đóng mặc định)**;
   `appends` chỉ một cấp và chỉ quan hệ được phép; `fields`, `filter`, `sort` chỉ trên cột được phép của bảng gốc và bảng đích
   (chặn "dò" dữ liệu qua lọc, ví dụ lọc `loi_giai` theo tiền tố); bản ghi nạp kèm bị cắt còn cột được phép.
6. `workflows:trigger` chỉ cho `root`, `admin`, `dsc_quan_ly`.
7. Thao tác ghi đặc biệt đi qua action của plugin, kiểm quyền ở server (làm bài, thi, xếp cặp, nhập kết quả, duyệt báo cáo AI).

Trên v3: dựng bằng permission set + data scope; **bắt buộc** mang theo bộ kiểm `parent-isolation.mjs` / `parent-oracle.mjs`
(hàng trăm phép dò) làm test chấp nhận, vì hành vi nạp kèm/quan hệ của v3 khác v2 và phải chứng minh lại.

## 4. Workflow (55)

Bản đủ (trigger, điều kiện, chuỗi node): JSON → `workflows[]`. Nhóm:

| Nhóm                      | Workflow                                                                                                                                                                                                                                                                   | Trigger                                 | Ghi chú v3                                                                                                           |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| Bộ máy tính toán (F0–F20) | Gói mới, điểm danh (trừ buổi, sao, sắp hết), xoá điểm danh, phiếu thu (đã thu, công nợ, số dư sách), sửa gói, giao dịch sao, phát sách, sửa học viên, làm bài, giao bài, lệ phí giải, tồn kho (dòng phiếu kho, phiếu kho), đơn hàng, phiếu thu bán hàng/nhà sách, đối soát | Tạo/sửa/xoá bản ghi (kèm bản "khi xoá") | Chuyển thành **service trong transaction** (tính lại ngay khi ghi), không cần workflow; giữ công thức `formulas.mjs` |
| Định kỳ                   | FN tính lại toàn bộ + hết hạn + sinh nhật (02:00), nhắc lead quá 24 h (08:30), ZNS nhận xét buổi học (20:30), báo tồn thấp (08:00)                                                                                                                                         | Lịch                                    | `scheduler`                                                                                                          |
| Nút bấm                   | FM tính lại toàn bộ; chuyển lead thành học viên + phụ huynh; ghi nhận phiếu thu từ giao dịch ngân hàng (ghép tay)                                                                                                                                                          | custom-action                           | Route/action tự viết                                                                                                 |
| Tuyển sinh                | Đăng ký web → lead; báo tư vấn lead mới                                                                                                                                                                                                                                    | Tạo bản ghi                             | Hook + notification                                                                                                  |
| Thi                       | Bài thi đủ điều kiện → chờ duyệt (báo quản lý); kết quả thi → báo phụ huynh (app + Zalo đạt cấp); đổi cấp tay → lịch sử cấp                                                                                                                                                | Sửa bản ghi                             | Hook + notification                                                                                                  |
| Giải đấu                  | Đăng ký web → báo; kỳ thủ xác nhận → báo phụ huynh (app + Zalo)                                                                                                                                                                                                            | Tạo/sửa                                 | Hook + notification                                                                                                  |
| Bán hàng                  | Đơn xác nhận/giao → phiếu xuất (một lần); đơn huỷ → huỷ phiếu xuất; trả hàng → phiếu nhập; lần in → phiếu nhập; phát sách → xuất kho CLB; đối soát xác nhận → xuất kho ký gửi; đơn web mới → báo                                                                           | Sửa trạng thái                          | Service có khoá chống trùng (`khoa_chong_trung`)                                                                     |
| Zalo/ngân hàng            | ZNS đã nhận học phí; báo giao dịch chờ xử lý                                                                                                                                                                                                                               | Tạo bản ghi                             | Hook                                                                                                                 |

Quy tắc dùng chung (`formulas.mjs` → `RULES`): có mặt = `co_mat`; trạng thái gói tay = `huy`, `het_han`; đang học = `dang_hoc`,
`sap_het`; báo sắp hết khi còn **2** buổi.

## 5. Giao diện

**Desktop (41 trang, 11 nhóm):** Tổng quan (bảng điều khiển: biểu đồ cột/tròn, nút tính lại) · Học viên (danh sách học viên,
phụ huynh) · Lớp & buổi học (lớp, buổi, điểm danh & nhận xét) · Học phí (đăng ký gói, phiếu thu, phát sách, giao dịch ngân hàng)
· Học liệu (bài học, thế cờ, giao bài, kết quả bài tập) · Thi cấp (đề, kỳ, bài thi, chứng nhận) · Giải đấu (giải, bảng, kỳ thủ,
vòng đấu, bảng xếp hạng, giấy khen) · Tuyển sinh (lead) · Bán hàng (tổng quan, đơn hàng, khách hàng, sản phẩm, kho & phiếu kho,
kho, lần in, nhà sách, đối soát ký gửi, nhập Shopee) · Thông báo Zalo (hàng đợi) · Danh mục (gói học phí, sách, nhân sự, cơ sở,
cấp độ). Mẫu chung mỗi trang: bộ lọc + bảng + thêm/xem/sửa trong popup chia nhóm trường; nhiều khối JS tự viết (bàn cờ, xếp
cặp, VietQR, hồ sơ, chứng nhận, giấy khen, biểu đồ Elo, nhập Shopee, phiếu giao hàng).

**Mobile `/v/mobile`:** giáo viên — Điểm danh, Tra cứu học viên, Sách, Giao bài, Phòng thi, Trọng tài; phụ huynh — Con tôi,
Nhận xét, Gói học phí, Bài tập, Bài học; bán hàng — Bán sách. Menu theo vai trò: `menuAccess` trong JSON.

**Công khai:** form "Đăng ký học thử" (plugin `public-forms`), trang đăng ký giải đấu + tra trạng thái + bảng xếp hạng/xếp cặp,
cửa hàng sách (danh mục, đặt hàng, tra đơn, `catalog.json`).

Trên v3: **toàn bộ giao diện phải viết lại** bằng React/shadcn (UI Library) — mảng tốn công nhất. Biểu đồ dùng thư viện (skill
`dataviz`), mobile làm giao diện responsive/PWA riêng cho giáo viên và phụ huynh.

## 6. Plugin `@duongsinh/*` và logic dùng lại được

| Plugin                   | Việc làm                                                               | Route                                     | Logic thuần mang sang được                     | Công sức v3         |
| ------------------------ | ---------------------------------------------------------------------- | ----------------------------------------- | ---------------------------------------------- | ------------------- |
| `dsc-guard` 0.1.8        | Cách ly phụ huynh, chặn workflow toàn cục                              | (middleware)                              | danh sách cột duyệt                            | M (chủ yếu là test) |
| `dsc-chess` 0.2.3        | Bài tập/thi thế cờ, chấm server, hồ sơ, chứng nhận                     | `dscChess:*` 17 action (đăng nhập)        | `grading.ts`                                   | L                   |
| `dsc-tournament` 0.3.1   | Xếp cặp Thuỵ Sĩ/vòng tròn, BXH, Elo, giấy khen, TRF16, trang công khai | 12 action đăng nhập + 5 công khai         | `core/{swiss,roundRobin,standings,elo,trf}.ts` | L                   |
| `dsc-shop` 0.4.0         | Tồn kho chặn âm, đơn, ký gửi, đối soát, Shopee, cửa hàng công khai     | 13 đăng nhập + 5 công khai                | `stock.ts`, parser Shopee                      | L                   |
| `dsc-bank-webhook` 0.3.0 | Webhook SePay → phiếu thu theo mã DSC/GD/DH                            | `POST dscBankWebhook:receive` (công khai) | `receive.ts`                                   | M                   |
| `dsc-zalo` 0.1.3         | Hàng đợi ZNS, hạn mức, làm mới token                                   | 3 action                                  | `zalo.ts`                                      | M                   |
| `dsc-ai` 0.1.0           | Nháp nhận xét/báo cáo tháng, che dữ liệu trẻ em                        | 4 action                                  | `privacy.ts`, `prompts.ts`                     | M                   |

## 7. Tích hợp ngoài (chỉ tên biến bí mật)

| Tích hợp             | Cách chạy                                                                                                                   | Bí mật                                                                           |
| -------------------- | --------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Lark Base → NocoBase | `setup/import-lark.mjs <thư mục ndjson>` (học viên, phụ huynh, lớp, đăng ký gói, nhận xét); khớp `lark_id`; không nhập CCCD | `NB_URL`, `NB_ACCOUNT`, `NB_PASSWORD`, `NB_TOKEN`                                |
| Zalo ZNS             | `business.openapi.zalo.me/message/template`; token qua `oauth.zaloapp.com/v4/oa/access_token`                               | `DSC_ZALO_APP_ID`, `DSC_ZALO_APP_SECRET`; token OA lưu trong `dsc_zalo_cau_hinh` |
| SePay                | `POST /api/dscBankWebhook:receive`, header `Authorization: Apikey`                                                          | `DSC_SEPAY_API_KEY`, `DSC_BANK_ACCOUNT`                                          |
| VietQR               | ảnh `img.vietqr.io/image/<NH>-<TK>-compact2.png?amount&addInfo`                                                             | `DSC_BANK_ACCOUNT`                                                               |
| Anthropic            | `api.anthropic.com/v1/messages` (chỉ khi bật chế độ thật)                                                                   | `DSC_AI_API_KEY`, `DSC_AI_PROVIDER`                                              |
| Shopee               | nhập file Excel đơn/thu nhập                                                                                                | —                                                                                |
| CDN trình duyệt      | `esm.sh/chess.js`, `cdn.jsdelivr.net` cm-chessboard, Google Fonts, YouTube embed                                            | —                                                                                |

Lưu ý khi chuyển: **URL webhook SePay** đang trỏ `https://nocobase.dsc.edu.vn/api/dscBankWebhook:receive` — app v3 phải giữ
đúng đường dẫn (hoặc đổi cấu hình bên SePay đúng lúc cắt chuyển); link form công khai đã gắn lên website/Facebook cũng vậy.

## 8. Tài sản kiểm chứng (thành test chấp nhận cho v3)

- `setup/verify.mjs` mục 1–16: bộ máy tính toán (tạo/sửa/xoá đều cập nhật), quyền từng vai trò nhân viên, cách ly phụ huynh
  (`parent-isolation.mjs`, `parent-oracle.mjs`), học liệu, thi, giải đấu, bán hàng (~80 phép), AI, Zalo, SePay, bundle client;
  `web-check.mjs` (đăng nhập, giao diện mobile).
- 22 file unit test `setup/tests/*.test.mjs` (Elo, xếp cặp, BXH, chấm thế cờ, VietQR, tồn kho, Shopee, che dữ liệu AI…) — chạy
  được không cần NocoBase, mang thẳng sang v3.
- CI PR #1: dựng NocoBase 2.2.36 + Postgres trong GitHub Actions — dùng làm **môi trường đối chứng** khi so kết quả v2/v3 trên
  cùng dữ liệu mẫu.

## 9. Thứ tự chuyển đề xuất và dữ liệu phải mang

**Thứ tự dựng trên v3** (mỗi bước có test chấp nhận + giao diện + công cụ chuyển dữ liệu của bước đó):

1. Nền: người dùng, vai trò, cơ sở, nhân sự, cấp độ, danh mục — **cách ly phụ huynh + bộ dò** ngay từ đầu.
2. Vận hành CLB (bản mẫu giai đoạn C): học viên, phụ huynh, lớp, buổi, điểm danh & nhận xét, thẻ sao — mobile giáo viên.
3. Học phí: gói, đăng ký gói, phiếu thu, phát sách, bộ máy tính toán, VietQR + webhook SePay.
4. Cổng phụ huynh (mobile phụ huynh) + tuyển sinh (lead, form công khai).
5. Học liệu + thi cấp (bàn cờ, chấm, chứng nhận).
6. Giải đấu (dùng lại core) + trang công khai.
7. Bán hàng & kho (dùng lại stock) + cửa hàng công khai + Shopee.
8. Zalo ZNS, AI (khi chủ dự án bật).

**Dữ liệu phải mang:** 49 bảng `dsc_*` + 8 bảng trung gian + `users` (và gán vai trò); không mang dữ liệu `[TEST]`, cấu hình
UI/workflow v2. Hai câu SQL để **chủ dự án tự chạy** trên production (chỉ đọc; tôi không chạy):

```sql
-- 1) Số dòng mọi bảng nghiệp vụ (PostgreSQL)
SELECT table_name AS bang,
       (xpath('/row/c/text()', query_to_xml(format('SELECT count(*) AS c FROM %I', table_name), false, true, '')))[1]::text::int AS so_dong
FROM information_schema.tables
WHERE table_schema = 'public' AND (table_name LIKE 'dsc\_%' OR table_name = 'users')
ORDER BY so_dong DESC;

-- 2) Mức dùng thật: số bản ghi tạo trong 30 ngày gần nhất và lần ghi gần nhất (đổi tên bảng cho từng mảng)
SELECT 'dsc_diem_danh' AS bang, count(*) FILTER (WHERE "createdAt" > now() - interval '30 days') AS moi_30_ngay, max("updatedAt") AS ghi_gan_nhat FROM dsc_diem_danh
UNION ALL SELECT 'dsc_phieu_thu', count(*) FILTER (WHERE "createdAt" > now() - interval '30 days'), max("updatedAt") FROM dsc_phieu_thu
UNION ALL SELECT 'dsc_lam_bai', count(*) FILTER (WHERE "createdAt" > now() - interval '30 days'), max("updatedAt") FROM dsc_lam_bai
UNION ALL SELECT 'dsc_ky_thu', count(*) FILTER (WHERE "createdAt" > now() - interval '30 days'), max("updatedAt") FROM dsc_ky_thu
UNION ALL SELECT 'dsc_don_hang', count(*) FILTER (WHERE "createdAt" > now() - interval '30 days'), max("updatedAt") FROM dsc_don_hang
UNION ALL SELECT 'dsc_giao_dich_ngan_hang', count(*) FILTER (WHERE "createdAt" > now() - interval '30 days'), max("updatedAt") FROM dsc_giao_dich_ngan_hang
UNION ALL SELECT 'dsc_zalo_hang_doi', count(*) FILTER (WHERE "createdAt" > now() - interval '30 days'), max("updatedAt") FROM dsc_zalo_hang_doi;
```

Bản liệt kê đủ từng bảng (không cần `query_to_xml`): JSON → `migration.rowCountSql`.

## 10. Rủi ro và thiếu hụt chính

1. **Giao diện viết lại toàn bộ** (41 + 12 trang, nhiều khối JS tự viết) — v3 không có trình dựng trang như v2.
2. **Cách ly phụ huynh** phải chứng minh lại trên cơ chế quyền mới; lỗi ở đây là lộ dữ liệu trẻ em.
3. **Không có gói v3** cho form công khai, mobile, biểu đồ, nhập/xuất Excel, đăng nhập SMS — phải tự làm.
4. **55 workflow chủ yếu là SQL** tính lại; dịch sang service cần làm cẩn thận để số liệu khớp (dùng `recalc.mjs` đối soát).
5. **Đường dẫn công khai đang dùng** (webhook SePay, form đăng ký, trang giải đấu, cửa hàng) phải giữ hoặc chuyển đồng bộ.
6. **Mã cũ và bộ đếm** (HV, PT, DH…) phải giữ nguyên, bộ đếm v3 nối tiếp.
7. **11 commit + phần AI chưa push** lên GitHub — production đang chạy mã chỉ có trên máy chủ dự án.
8. v3 beta đổi API hằng tuần — mọi phần viết lại có thể phải sửa theo.
9. VPS thiếu RAM để chạy song song v2 + v3 + staging.
10. Dữ liệu cá nhân trẻ em trong mọi lần chạy thử chuyển dữ liệu — chỉ dùng bản đã che.

## 11. Câu hỏi cho chủ dự án

1. Chạy hai câu SQL mục 9 trên production và gửi kết quả (chỉ số đếm, không gửi dữ liệu) để biết mảng nào đang dùng thật.
2. Mảng nào **bỏ được** khi chuyển (ví dụ Zalo chưa dùng, AI dở dang, Lark import đã xong)?
3. Giáo viên/phụ huynh hiện dùng mobile `/v/mobile` thật chưa? Có cần ứng dụng cài đặt (PWA) không?
4. Webhook SePay và form công khai đã nhận dữ liệu thật chưa?
5. Đồng ý push 11 commit đang chỉ có trên máy (sau khi rà bí mật) để mã production có bản trên GitHub?
6. Giữ nguyên tên miền/đường dẫn công khai khi cắt chuyển, hay chấp nhận đổi?
