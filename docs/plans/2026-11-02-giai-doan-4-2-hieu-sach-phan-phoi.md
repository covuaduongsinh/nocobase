# Kế hoạch 4.2 — Hiệu sách, học cụ và phân phối sách (web, fanpage, Shopee, nhà sách)

Thuộc kế hoạch giai đoạn 4 `docs/plans/2026-11-02-giai-doan-4-mo-rong.md`. Lập ngày 02/11/2026.

## 1. Bối cảnh và yêu cầu

**Rà soát:** 4.1 Giải đấu đã xong (commit `0bdf2f7`, plugin `dsc-tournament` 0.3.0, verify production 592 phép đạt).

**Hiện trạng bán sách trong hệ thống:**

- Danh mục `dsc_sach` có 37 đầu sách (ký hiệu, giá bìa, giá bán cho phụ huynh).
- `dsc_phat_sach` có 338 lần phát sách cho học viên.
- Có phiếu thu loại `tien_sach` và số dư tiền sách của học viên.
- **Chưa có:** tồn kho, đơn hàng theo kênh, khách lẻ, nhà sách, ký gửi.

**Yêu cầu của chủ dự án (02/11/2026):** bán sách và học cụ qua 4 kênh, và phân phối sách do Dương Sinh sản xuất / xuất bản / phát hành cho các nhà sách lớn.

- **Web** (covuaduongsinh.com)
- **Fanpage** (Facebook)
- **Shopee**. Tiki đã ngừng bán nên không làm.
- **Nhà sách lớn**

## 2. Giả định mặc định

Chủ dự án đổi được sau.

| #   | Điểm                   | Mặc định                                                                                                                                                                                                                                                                                                                                        |
| --- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| S1  | Shopee                 | **Nhập file xuất đơn hàng** từ Shopee Seller Center (CSV/XLSX), khớp theo mã đơn Shopee, chạy lại không trùng. Đối soát tiền về từ **báo cáo doanh thu / thanh toán** của Shopee (phí sàn, phí vận chuyển, số thực nhận). **Không dùng Shopee Open API** ở bản này, vì cần đăng ký đối tác và duyệt app; có thể làm sau                         |
| S2  | Web                    | **Trang cửa hàng công khai do plugin phục vụ** (giống trang đăng ký giải): danh mục, giỏ hàng, đặt hàng, VietQR. covuaduongsinh.com chỉ cần đặt liên kết hoặc nút "Mua sách". Có thêm **JSON danh mục công khai** để website tự hiển thị nếu muốn                                                                                               |
| S3  | Fanpage                | Nhân viên nhập nhanh đơn từ tin nhắn (form "Đơn nhanh" trên điện thoại và máy tính). Kèm **link cửa hàng có `?nguon=fanpage`** để đăng lên fanpage, đơn tự gắn kênh. Không tích hợp Messenger API                                                                                                                                               |
| S4  | Thanh toán             | VietQR, nội dung chuyển khoản `DSC DH#####`. Webhook SePay mở rộng thêm mã `DH`. Hỗ trợ COD (thu hộ, đối soát với đơn vị vận chuyển bằng tay) và tiền mặt tại quầy                                                                                                                                                                              |
| S5  | Kho                    | Các kho: **Trụ sở / hiệu sách Tôn Đức Thắng** (kho chính), **CLB Kim Liên**, **CLB Vĩnh Phúc**. Mỗi nhà sách nhận ký gửi có **một kho ký gửi riêng** (hàng vẫn thuộc Dương Sinh). Tồn kho = tổng nhập − xuất theo từng kho, tự tính                                                                                                             |
| S6  | Nhà sách               | Hai hình thức: **mua đứt** (giao hàng → ghi công nợ ngay theo chiết khấu) và **ký gửi** (chuyển hàng sang kho ký gửi → nhà sách báo bán định kỳ → đối soát → ghi công nợ phần đã bán; nhận lại hàng tồn hoặc trả). Chiết khấu % theo đối tác, ghi đè được theo từng đầu sách                                                                    |
| S7  | Xuất bản               | Đầu sách tự xuất bản có thêm thông tin xuất bản (ISBN, NXB liên kết, quyết định xuất bản, tác giả). Mỗi **lần in** (số lượng, nhà in, chi phí, ngày nhận) tự sinh phiếu nhập kho chính. **Nhuận bút và giá vốn chi tiết chưa làm ở bản này** (chỉ lưu chi phí in để tham khảo)                                                                  |
| S8  | Hoá đơn GTGT           | Làm trên phần mềm hoá đơn điện tử hiện có. Hệ thống chỉ **lưu số / ký hiệu hoá đơn** trên đơn hàng hoặc phiếu đối soát                                                                                                                                                                                                                          |
| S9  | Phát sách cho học viên | Từ nay mỗi lần phát sách **xuất kho của CLB**. **Không truy thu** 338 lần phát cũ: tồn đầu kỳ nhập bằng **phiếu kiểm kê đầu kỳ**                                                                                                                                                                                                                |
| S10 | Vai trò mới            | `dsc_ban_hang` (bán hàng): đơn hàng, khách hàng, xem tồn kho, phiếu xuất của đơn. **Kế toán:** phiếu thu, công nợ, đối soát nhà sách / Shopee. **Quản lý:** toàn quyền. **Giáo viên:** chỉ xem tồn kho của CLB mình và phát sách như cũ. **Phụ huynh:** không thấy gì của mảng này, trừ đơn của chính mình nếu sau này nối tài khoản (chưa làm) |

## 3. Dữ liệu (`setup/schema/phase4-shop.mjs`)

| Collection           | Trường chính                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Quan hệ                                                                                       |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| `dsc_san_pham`       | `ma` (SKU, unique), `ten`, `loai` (`sach` / `hoc_cu` / `khac`), `gia_bia`, `gia_ban_le`, `don_vi`, `dsc_phat_hanh` (checkbox: Dương Sinh xuất bản/phát hành), `isbn`, `tac_gia`, `nxb_lien_ket`, `so_qdxb`, `mo_ta`, `anh` (attachment), `ban_online` (checkbox), `shopee_sku`, `muc_ton_toi_thieu`, `ton_kho_tong` (tự tính), `trang_thai` (`dang_ban` / `ngung`)                                                                                                                 | m2o `sach` (`dsc_sach`, 1-1 cho 37 đầu sách cũ)                                               |
| `dsc_kho`            | `ma`, `ten`, `loai` (`chinh` / `clb` / `ky_gui`)                                                                                                                                                                                                                                                                                                                                                                                                                                   | m2o `co_so`, m2o `doi_tac` (khi ký gửi)                                                       |
| `dsc_ton_kho`        | `so_luong` (tự tính)                                                                                                                                                                                                                                                                                                                                                                                                                                                               | m2o `kho`, m2o `san_pham` (duy nhất theo cặp)                                                 |
| `dsc_phieu_kho`      | `so` (sequence `PK`), `loai` (`nhap_in` / `nhap_mua` / `nhap_tra` / `xuat_ban` / `xuat_phat_sach` / `chuyen` / `kiem_ke` / `xuat_huy`), `ngay`, `ghi_chu`, `trang_thai` (`nhap` / `da_ghi_so` / `huy`)                                                                                                                                                                                                                                                                             | m2o `kho_xuat`, m2o `kho_nhap`, m2o `don_hang`, m2o `lan_in`, m2o `doi_soat`, m2o `phat_sach` |
| `dsc_phieu_kho_dong` | `so_luong`, `don_gia`                                                                                                                                                                                                                                                                                                                                                                                                                                                              | m2o `phieu_kho`, m2o `san_pham`                                                               |
| `dsc_khach_hang`     | `ho_ten`, `dien_thoai`, `dia_chi`, `email`, `ghi_chu`, `nguon`                                                                                                                                                                                                                                                                                                                                                                                                                     | m2o `phu_huynh` (tuỳ chọn, khi trùng SĐT)                                                     |
| `dsc_don_hang`       | `ma` (sequence `DH`), `kenh` (`web` / `fanpage` / `shopee` / `tai_quay` / `nha_sach` / `hoc_vien`), `ma_don_ngoai` (mã Shopee), `ngay`, `trang_thai` (`moi` / `xac_nhan` / `dang_giao` / `hoan_thanh` / `huy` / `tra_hang`), `thanh_toan` (`vietqr` / `chuyen_khoan` / `cod` / `tien_mat` / `shopee` / `cong_no`), `phi_ship`, `giam_gia`, `tong_hang` / `tong_tien` / `da_thu` / `con_no` (tự tính), `dia_chi_giao`, `ma_van_don`, `so_hoa_don`, `phi_san` / `thuc_nhan` (Shopee) | m2o `khach_hang`, m2o `doi_tac`, m2o `kho`                                                    |
| `dsc_don_hang_dong`  | `so_luong`, `don_gia`, `chiet_khau_pt`, `thanh_tien` (tự tính)                                                                                                                                                                                                                                                                                                                                                                                                                     | m2o `don_hang`, m2o `san_pham`                                                                |
| `dsc_doi_tac`        | `ma`, `ten` (Fahasa, Phương Nam…), `mst`, `dia_chi`, `nguoi_lien_he`, `dien_thoai`, `chiet_khau_pt`, `hinh_thuc` (`mua_dut` / `ky_gui` / `ca_hai`), `han_thanh_toan_ngay`, `cong_no` (tự tính)                                                                                                                                                                                                                                                                                     | —                                                                                             |
| `dsc_doi_soat`       | `so` (sequence `DS`), `ky_tu`, `ky_den`, `trang_thai` (`nhap` / `da_xac_nhan` / `da_thanh_toan`), `tong_tien` (tự tính), `so_hoa_don`                                                                                                                                                                                                                                                                                                                                              | m2o `doi_tac`, m2o `kho` (kho ký gửi)                                                         |
| `dsc_doi_soat_dong`  | `so_luong_ban`, `don_gia`, `chiet_khau_pt`, `thanh_tien`                                                                                                                                                                                                                                                                                                                                                                                                                           | m2o `doi_soat`, m2o `san_pham`                                                                |
| `dsc_lan_in`         | `lan_thu`, `so_luong`, `nha_in`, `chi_phi`, `ngay_nhan`, `ghi_chu`                                                                                                                                                                                                                                                                                                                                                                                                                 | m2o `san_pham`, m2o `kho` (mặc định kho chính)                                                |

- `dsc_phieu_thu.loai_thu` thêm `ban_hang` và `nha_sach`. Thêm m2o `don_hang` và `doi_tac`.
- `dsc_phat_sach` thêm m2o `kho` (mặc định kho của CLB học viên).

## 4. Bộ máy tính toán

Đưa vào `formulas.mjs` + `recalc.mjs` + FN/FM, đúng nguyên tắc "một công thức, tự cập nhật, kiểm toán độc lập".

- **Tồn kho** `dsc_ton_kho.so_luong` theo (kho, sản phẩm) = Σ dòng phiếu `da_ghi_so` nhập vào kho − Σ dòng xuất khỏi kho. Riêng `kiem_ke` đặt mức tuyệt đối: lưu dòng chênh lệch khi ghi sổ.
- `dsc_san_pham.ton_kho_tong` = Σ tồn ở các kho **không phải** ký gửi. Hàng đang ký gửi hiện ở cột riêng.
- **Đơn hàng:**
  - `thanh_tien` dòng = số lượng × đơn giá × (1 − chiết khấu);
  - `tong_hang` = Σ dòng; `tong_tien` = tổng hàng + phí ship − giảm giá;
  - `da_thu` = Σ phiếu thu `ban_hang` của đơn;
  - `con_no` = tổng tiền − đã thu (Shopee thì dựa vào `thuc_nhan`).
- **Công nợ đối tác** = Σ đơn `nha_sach` mua đứt đã giao + Σ đối soát ký gửi đã xác nhận − Σ phiếu thu `nha_sach` của đối tác.

**Workflow:**

- **Đơn hàng sang `xac_nhan` / `dang_giao`:** tạo phiếu `xuat_ban` từ kho của đơn và ghi sổ (một lần, có khoá chống trùng). Đơn chuyển `huy` thì huỷ phiếu xuất nếu chưa giao. Đơn chuyển `tra_hang` thì tạo phiếu `nhap_tra`.
- **Lần in được tạo:** tạo phiếu `nhap_in` vào kho đã chọn.
- **Phát sách được tạo** (từ nay về sau): tạo phiếu `xuat_phat_sach` từ kho của CLB.
- **Đối soát ký gửi được xác nhận:** tạo phiếu xuất khỏi kho ký gửi theo số lượng đã bán.
- **Tồn dưới mức tối thiểu:** báo quản lý trong app, mỗi ngày một lần.
- **Không cho xuất âm kho:** kiểm ở plugin. Quản lý được phép ghi đè, và mỗi lần ghi đè đều được lưu lại.

## 5. Plugin mới `@duongsinh/plugin-dsc-shop`

Khung lấy từ `dsc-tournament`. Thư mục gốc có `client.js` / `server.js`.

**Trang cửa hàng công khai:**

- `GET dscShop:page?nguon=` trả trang HTML tự chứa: danh mục `ban_online` còn hàng, ảnh, giá, giỏ hàng lưu ở localStorage.
- `POST dscShop:order` nhận đơn:
  - họ tên, SĐT, địa chỉ, ghi chú, chọn VietQR hoặc COD;
  - bắt buộc đồng ý xử lý dữ liệu; có trường bẫy (honeypot) và giới hạn tần suất;
  - kiểm tồn kho, tạo khách hàng (ghép theo SĐT) và đơn `moi`, kênh `web` hoặc `fanpage` theo tham số `nguon`;
  - trả mã `DH#####` kèm VietQR.
- `GET dscShop:status?ma=&sdt=` để tra cứu đơn.
- `GET dscShop:catalog.json` trả danh mục công khai: tên, giá, ảnh, còn hàng hay không. Không trả số tồn chính xác.

**Nhập Shopee:**

- `dscShop:importShopee`, chỉ quản lý và kế toán được gọi.
- Nhận file đơn hàng CSV/XLSX; dùng thư viện đọc XLSX nhẹ, giấy phép MIT/Apache, kiểm tra trước khi dùng.
- Map theo `shopee_sku` hoặc `ma`. Cập nhật trạng thái theo cột trạng thái của Shopee. Chạy lại không tạo trùng.
- Trả báo cáo: dòng lỗi, SKU lạ.
- Có thêm `importShopeeIncome` để nhập báo cáo doanh thu (phí sàn, thực nhận theo từng đơn).
- **Implementer phải lấy định dạng cột thật** từ file mẫu do chủ dự án xuất. Nếu chưa có file mẫu thì viết bộ map cấu hình được, kèm tài liệu "cột nào nghĩa là gì", và ghi vào mục "PHẢI xác minh".

**Nghiệp vụ nội bộ:**

- `quickOrder`: đơn nhanh cho fanpage hoặc bán tại quầy.
- `transfer`: chuyển kho, gồm cả gửi hàng ký gửi cho nhà sách.
- `stocktake`: kiểm kê đầu kỳ và định kỳ.
- `settle`: tạo đối soát ký gửi từ báo cáo bán của nhà sách, nhập tay hoặc từ CSV.

**Mở rộng `dsc-bank-webhook`:** tách thêm mã `DH\s?(\d{5})`, tạo phiếu thu `ban_hang` gắn đơn hàng. Nếu nội dung có nhiều loại mã (`DK` / `GD` / `DH`) thì chuyển sang `cho_xu_ly`.

## 6. Giao diện

**Desktop**, nhóm mới "Bán hàng":

- Đơn hàng: lọc theo kênh / trạng thái; nút đổi trạng thái; in phiếu giao hàng A5.
- Khách hàng.
- Sản phẩm: tồn theo từng kho, phần "Lần in".
- Kho & phiếu kho: tạo phiếu nhập, xuất, chuyển, kiểm kê.
- Nhà sách: đối tác, giao mua đứt, gửi ký gửi, đối soát, công nợ.
- Nhập Shopee: khối tải file lên, xem trước, xác nhận.

**Dashboard:**

- doanh thu theo kênh và theo tháng;
- 10 sản phẩm bán chạy;
- tồn thấp;
- hàng đang ký gửi ở từng nhà sách;
- công nợ nhà sách theo tuổi nợ.

**Mobile bán hàng / giáo viên:** trang "Bán sách" để nhập đơn tại quầy hoặc CLB trên điện thoại:

- tìm sản phẩm, chọn số lượng;
- tiền mặt hoặc VietQR; kho mặc định là CLB của người dùng.
- Giáo viên đang dùng 6 tab. Gộp "Phát sách" và "Bán sách" thành **"Sách"** gồm 2 khối, để không vượt quá 6 tab.

## 7. Thứ tự triển khai

1. **4.2-1** Schema + ACL (vai trò `dsc_ban_hang`) + seed kho. Chuyển 37 đầu sách thành sản phẩm (giữ liên kết `dsc_sach`). Sinh file mẫu kiểm kê đầu kỳ `reports/5-kiem-ke-dau-ky.csv`. Bộ dò (isolation, oracle, staff).
2. **4.2-2** Bộ máy tính tồn kho, đơn hàng, công nợ (`formulas.mjs` + test SQLite/PG + `recalc`). Workflow đơn ↔ phiếu kho, lần in, phát sách, đối soát, tồn thấp. Chặn xuất âm kho.
3. **4.2-3** Plugin `dsc-shop`: cửa hàng công khai, catalog JSON, tra cứu, webhook `DH`, đơn nhanh, chuyển kho, kiểm kê.
4. **4.2-4** Nhập Shopee (đơn + doanh thu). Nhà sách: mua đứt, ký gửi, đối soát, công nợ.
5. **4.2-5** Giao diện desktop "Bán hàng", dashboard, mobile "Sách", in phiếu giao hàng. Playwright. Ghi mục "Kết quả".

Mỗi bước: dev (ít nhất unit test và các phép kiểm nhẹ) → sao lưu → production → verify đầy đủ + plugins + web-check → dọn `[TEST]` và đặt lại bộ đếm số (`PK`, `DH`, `DS`) → commit. Giữ mọi quy tắc đã có: tài khoản thử theo từng vai trò, `collections` kéo theo `roles`, `gen-parent-columns`, không dùng tài khoản thật, không gửi Zalo.

## 8. Kiểm chứng (`verify.mjs` mục 15, `setup/shop-check.mjs`)

1. **Kho:** kiểm kê đầu kỳ đặt tồn → nhập lần in +500 → đơn web 3 cuốn xác nhận thì tồn −3 → đơn huỷ trước khi giao thì tồn +3 → trả hàng thì +3. `recalc --check` = 0 lệch.
2. **Xuất âm:** bị chặn (400). Quản lý ghi đè được và có ghi lại.
3. **Cửa hàng công khai:**
   - đặt hàng hợp lệ → mã `DH` + QR;
   - hết hàng → 400; honeypot → 200 nhưng không tạo đơn; quá giới hạn tần suất → 429;
   - `catalog.json` không có số tồn chính xác hay dữ liệu nội bộ;
   - tra cứu sai SĐT → 404.
4. **Webhook `DH`** (khoá thử trên dev): phiếu thu `ban_hang`, đơn hết nợ; gửi lại không trùng; nội dung có nhiều loại mã → `cho_xu_ly`; mọi ca `DK` / `GD` cũ vẫn đạt.
5. **Shopee:** nhập file mẫu 10 đơn → đúng 10 đơn kênh `shopee`; nhập lại → không trùng; SKU lạ → báo dòng lỗi; nhập doanh thu → `phi_san` / `thuc_nhan` đúng.
6. **Nhà sách:**
   - mua đứt 100 cuốn, chiết khấu 30% → công nợ đúng;
   - ký gửi 200 cuốn → chuyển kho ký gửi;
   - đối soát bán 50 → xuất khỏi kho ký gửi, công nợ tăng đúng 50 × giá × (1 − CK);
   - thanh toán → công nợ giảm.
7. **Phát sách mới** xuất đúng kho CLB. 338 lần phát cũ không bị đụng.
8. **Quyền:**
   - bán hàng tạo đơn được, không sửa công nợ hay đối soát được;
   - giáo viên không xem được đơn hàng, chỉ xem tồn CLB;
   - kế toán đối soát được;
   - phụ huynh không thấy mọi bảng mới; bộ dò cách ly và oracle đạt.
9. **Trình duyệt** (Playwright, desktop + iPhone 13): đặt hàng qua cửa hàng công khai → nhân viên xác nhận đơn → in phiếu giao → bán tại CLB trên điện thoại → nhập file Shopee → đối soát ký gửi → dashboard. 0 lỗi JS.

## 9. Rủi ro

- **Sai tồn kho khi dùng thật:** dùng một công thức duy nhất, có kiểm toán `recalc`, chặn xuất âm, kiểm kê định kỳ, và mọi phiếu đều để lại vết.
- **Định dạng file Shopee thay đổi:** bộ map cột cấu hình được, báo dòng lỗi thay vì nhập sai.
- **Lộ dữ liệu khách hàng:** phụ huynh và giáo viên không có quyền; trang công khai chỉ trả dữ liệu tối thiểu; có giới hạn tần suất.
- **Cửa hàng công khai bị spam đơn ảo:** honeypot, giới hạn tần suất, đơn `moi` chưa trừ kho cho tới khi nhân viên xác nhận.
- **Nhầm mã thanh toán giữa các loại:** mỗi loại một tiền tố (`DK` / `GD` / `DH`); có nhiều loại mã thì xử lý tay.

## 10. Việc của chủ dự án

- Xác nhận các giả định S1–S10, nhất là danh sách kho, vai trò `dsc_ban_hang` và chiết khấu của từng nhà sách.
- Xuất **1 file đơn hàng** và **1 file doanh thu** mẫu từ Shopee Seller Center (có thể che SĐT và địa chỉ) để khớp đúng định dạng cột.
- Danh sách nhà sách đối tác: tên, MST, chiết khấu, hình thức (mua đứt / ký gửi).
- Kiểm kê tồn đầu kỳ theo file mẫu.
- Ảnh sản phẩm và các đầu sách / học cụ bán online.

## 11. Kết quả (02/11/2026)

### 4.2-1 + 4.2-2 Schema, ACL, bộ máy tồn kho / đơn hàng / công nợ, plugin `@duongsinh/plugin-dsc-shop` 0.1.0 — xong, dev + production

- **Schema** `setup/schema/phase4-shop.mjs` (12 bảng; sequence `PK` 2026108, `DH` 2026109, `DS` 2026110) + `dsc_phieu_thu.loai_thu` thêm `ban_hang` / `nha_sach` kèm `don_hang`, `doi_tac` + `dsc_phat_sach.kho`. `run.mjs shop` tạo 3 kho (K-TDT chính, K-KL, K-VP), chuyển **37 đầu sách → 37 sản phẩm** (giữ liên kết `sach`, chạy lại an toàn) và ghi `reports/5-kiem-ke-dau-ky.csv` (3 kho × 37 sản phẩm, để chủ dự án đếm kho). Production hiện: 0 phiếu kho, 0 đơn, tồn 0 — chờ kiểm kê đầu kỳ.
- **ACL**: vai trò mới `dsc_ban_hang` (đơn / dòng / khách tạo + sửa theo `fields` tường minh; chỉ xem sản phẩm, kho, tồn, phiếu kho; không đối soát, không phiếu thu). Quản lý toàn quyền; kế toán xem + sửa đối soát và chỉ `so_hoa_don` / `phi_san` / `thuc_nhan` / `ghi_chu` của đơn; giáo viên xem sản phẩm và tồn kho CLB mình; tư vấn và phụ huynh không có quyền nào (bộ dò cách ly + staff đạt).
- **Công thức** (`formulas.mjs` khối 8, một nguồn cho workflow + FN / FM + `recalc.mjs`): tồn = Σ phiếu `da_ghi_so` nhập − xuất (kiểm kê là dòng có dấu), `ton_kho_tong` không tính kho ký gửi, `ton_ky_gui` riêng; thành tiền = ROUND(SL × giá × (100 − CK)/100); tổng = hàng + ship − giảm; đã thu = Σ phiếu thu `ban_hang` (Shopee: thực nhận + phí sàn); công nợ nhà sách = đơn đã giao + đối soát xác nhận − phiếu thu `nha_sach`. Test fixture tính tay **đạt trên SQLite và PostgreSQL thật** (schema tạm + ROLLBACK).
- **Workflow** `setup/workflows/shop.mjs`: F12–F20 (số liệu), W20 đơn xác nhận / giao → phiếu xuất MỘT lần (khoá `xuat_ban:<đơn>`), W21 huỷ → huỷ phiếu, W22 trả hàng → nhập lại, W23 lần in → phiếu nhập, W24 phát sách MỚI (`lark_id` trống) → xuất kho CLB, W25 đối soát xác nhận → xuất kho ký gửi, W26 tồn thấp 08:00. Phiếu không ghi sổ được (thiếu tồn) giữ `nhap` và báo quản lý.
- **Plugin `dsc-shop`**: hook `dsc_phieu_kho.beforeSave` chặn ghi sổ làm tồn kho xuất âm (HTTP 400); quản lý ghi đè bằng `cho_phep_am` + `ly_do_ghi_de`, plugin ghi `nguoi_ghi_de` (id + giờ); người khác 403; phiếu đã ghi sổ chỉ được huỷ, dòng bị khoá. 8 test lõi thuần.
- **Kiểm chứng**: `verify.mjs` mục 15 (`setup/shop-check.mjs`, ~80 phép: kho 15.1, xuất âm 15.2, nhà sách + ký gửi + đối soát 15.6, phát sách mới 15.7, quyền 15.8) đạt trên dev và production; dev: formulas, shop-stock, guard, staff, parents đạt; production: `verify` đầy đủ chỉ trượt 1 phép recalc toàn cục do dữ liệu `[TEST]` của mục thi cấp đang dở (đã giới hạn phép này vào bảng bán hàng và chạy lại mục 15: đạt), `verify plugins` + `web-check` đạt, `recalc` 0 lệch, 338 lần phát sách cũ không có phiếu kho.

### Lệch kế hoạch / quyết định cần chủ dự án biết

1. `xuat_phat_sach` KHÔNG bị chặn xuất âm (chưa kiểm kê đầu kỳ thì tồn là 0, không được chặn giáo viên phát sách); thiếu hụt lộ ra ở báo cáo tồn.
2. Ảnh sản phẩm dùng `anh_url` (URL), chưa dùng tệp đính kèm.
3. Đơn huỷ rồi xác nhận lại: ghi sổ lại phiếu xuất cũ (không tạo phiếu mới, khoá chống trùng).
4. Chiết khấu dòng để trống = theo đối tác (đơn `nha_sach`), khác thì 0.
5. Phát sách mới bằng sách có liên kết sản phẩm sẽ trừ kho CLB thật ngay từ bây giờ (đúng thiết kế); các mục verify khác từng sinh phiếu mồ côi nên `verify.mjs` có bước `cleanOrphanIssueSlips`.
6. (Đã làm ở các bước 4.2-3 → 4.2-5, xem bên dưới.)

### 4.2-3 → 4.2-4 — cửa hàng công khai, webhook `DH`, nhập Shopee, nhà sách — xong (02/11/2026)

Plugin `dsc-shop` 0.2.0 → 0.3.2: cửa hàng công khai + catalog JSON + tra cứu, webhook mã `DH`, đơn nhanh / chuyển kho / kiểm kê, nhập Shopee (đơn + doanh thu, bộ map cột cấu hình — tên cột là GIẢ ĐỊNH, xem README), nhà sách (mua đứt, ký gửi, đối soát, công nợ). Commit `9348e05`.

### 4.2-5 Giao diện "Bán hàng", dashboard, mobile "Sách" / "Bán sách", in phiếu giao — xong, dev + production (03/11/2026)

- **Plugin `dsc-shop` 0.4.0** (commit `14916bf`): `shop-ops.ts` (báo cáo `shopReport`, thao tác giao diện) + 6 unit test; nhóm desktop **Bán hàng** 10 trang (Tổng quan bán hàng, Đơn hàng, Khách hàng, Sản phẩm, Kho & phiếu kho, Kho, Lần in, Nhà sách, Đối soát ký gửi, Nhập Shopee); in phiếu giao A5 (iframe, địa chỉ thoát HTML); dashboard doanh thu theo tháng / kênh, top 10, tồn thấp, hàng ký gửi, công nợ theo tuổi nợ.
- **Mobile:** gộp "Phát sách" + "Bán sách" thành tab **Sách** (giáo viên vẫn 6 tab); vai trò `dsc_ban_hang` có trang riêng **Bán sách**. Trang mobile "Phát sách" cũ đã xoá khỏi production (chỉ trang, không dữ liệu; 338 lần phát sách còn nguyên).
- **Production:** sao lưu `db_2026-11-03_0313` (+ gdrive) → cài 0.4.0 → `run.mjs ui` (lần đầu bị dừng do giới hạn thời gian nền; chạy lại chỉ các trang đổi bằng `DSC_PAGES`) → `verify.mjs` đầy đủ **846 phép đạt**, `verify.mjs plugins` + `web-check.mjs` đạt → Playwright kịch bản 8.9 (`setup/ui-shots-ban-hang.mjs`, ảnh `reports/production/ban-hang-*.png`, 12 ảnh, 28 phép đạt, 0 lỗi JS / API) → `audit-test-data.mjs` 0 dòng thử, `recalc.mjs` 0 lệch, bộ đếm `PK` / `DH` / `DS` đặt lại.
- **Bẫy mới:** (a) áp cả bộ UI trên production > 10 phút → dùng `DSC_PAGES="tiêu đề 1,tiêu đề 2"`; (b) `verify.mjs` đầy đủ dùng hết hạn mức 5 đơn / 10 phút / IP của cửa hàng công khai → chờ ~11 phút rồi mới chạy Playwright (không nới giới hạn); (c) `verify.mjs` để lại 5 phiếu kho `xuat_phat_sach` mồ côi + tồn −5 vì `cleanOrphanIssueSlips` chạy TRƯỚC khi xoá các lần phát sách thử → đã sửa thứ tự (chạy sau vòng xoá) và dọn thủ công; (d) `ton_kho_tong` còn −1 sau khi xoá phiếu → `recalc.mjs --apply`.
- **Việc của chủ dự án:** đếm kho và nhập phiếu kiểm kê đầu kỳ (`reports/5-kiem-ke-dau-ky.csv`); xác minh tên cột file Shopee thật; xem thử giao diện trên thiết bị thật.
