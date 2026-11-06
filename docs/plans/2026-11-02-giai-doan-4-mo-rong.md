# Kế hoạch giai đoạn 4 — Mở rộng: Giải đấu, Hiệu sách, AI, Trường đối tác, Nhượng quyền

Thuộc kế hoạch chiến lược `docs/plans/2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md` (giai đoạn 4). Lập ngày 02/11/2026.

## 1. Bối cảnh và rà soát

**Đã xong đến 02/11/2026:**

- Giai đoạn 1.
- Giai đoạn 2, bước 2.1–2.5. Zalo và SePay đang tắt, chờ khoá từ chủ dự án.
- Giai đoạn 3, bước 3.1–3.4: học liệu, bàn cờ, bài tập về nhà, thi lên cấp, chứng nhận, hồ sơ năng lực. Commit cuối `7db3df6`. Plugin `dsc-chess` 0.2.3, `dsc-guard` 0.1.6.

**Hoãn lại:**

- 2.6 kích hoạt phụ huynh: làm khi bắt đầu dùng thật.
- 3.5–3.7 (hợp nhất app cũ, `kids-player`, khoá online trả phí): theo lựa chọn của chủ dự án ngày 02/11/2026.

**Chủ dự án (02/11/2026):** chuyển sang giai đoạn 4, lập kế hoạch rồi giao triển khai luôn. Hệ thống "dùng thật sau".

**Thứ tự các mảng giai đoạn 4:**

| Mảng                                          | Nội dung                                                                                                                                                                                                                  | Lý do thứ tự                                                                                               |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| **4.1 Giải đấu** (lượt này, chi tiết ở mục 3) | Đăng ký online, lệ phí qua VietQR/SePay, xếp cặp Thuỵ Sĩ / vòng tròn, nhập kết quả, bảng xếp hạng công khai, Elo nội bộ, giấy khen                                                                                        | Giải nội bộ hằng quý và DSC Open là hoạt động sẵn có. Dùng lại học viên, phụ huynh, thanh toán, chứng nhận |
| 4.2 Hiệu sách & học cụ                        | Sản phẩm (mở rộng `dsc_sach`), tồn kho, đơn hàng tại quầy/online, nhập đơn Shopee/Tiki định kỳ (CSV), báo cáo                                                                                                             | Đã có danh mục 37 sách và phát sách                                                                        |
| 4.3 AI                                        | `plugin-ai`: soạn nhận xét từ ghi chú ngắn, tóm tắt tiến bộ tháng cho phụ huynh (dựa trên `dscChess:profile`), gợi ý bài tập theo điểm yếu. `plugin-mcp-server` để Claude truy vấn dữ liệu vận hành (chỉ vai trò quản lý) | Cần khoá nhà cung cấp AI và chính sách dữ liệu trẻ em                                                      |
| 4.4 Trường đối tác                            | Trường, hợp đồng, lớp ngoại khoá, báo cáo cho nhà trường                                                                                                                                                                  | Dùng lại lớp / buổi / điểm danh                                                                            |
| 4.5 Nhượng quyền                              | `plugin-multi-app-manager`: mỗi đối tác một sub-app, học liệu dùng chung, báo cáo tổng                                                                                                                                    | Lớn nhất, làm cuối                                                                                         |

Mỗi mảng 4.2–4.5 sẽ có file kế hoạch chi tiết riêng khi tới lượt.

**Tiến độ (03/11/2026):** 4.1 Giải đấu xong; 4.2 Hiệu sách & phân phối xong (`2026-11-02-giai-doan-4-2-hieu-sach-phan-phoi.md` mục 11, plugin `dsc-shop` 0.4.0); 4.3–4.5 chưa làm.

## 2. Ràng buộc chung (rút ra từ các sự cố 01/11)

- Mã riêng ở `D:\code\duongsinh-nocobase`, không sửa lõi NocoBase.
- Mọi thay đổi bằng script chạy lại an toàn. Mỗi bước: dev → verify → sao lưu → production → verify + `verify.mjs plugins` + `web-check.mjs` → commit.
- Kiểm bằng **tài khoản thử của từng vai trò**, không chỉ bằng root.
- Plugin phải có `client.js` / `server.js` ở thư mục gốc.
- `collections` luôn chạy kèm `roles`. Thêm trường cho phụ huynh xem thì sửa `apply.mjs` rồi chạy `gen-parent-columns` (guard đóng mặc định).
- Không truyền `transaction` vào `Repository.find` có `appends`.
- Không dùng tài khoản thật. Dọn dữ liệu `[TEST]` và đặt lại các bộ đếm số (`sequence.mjs`) sau khi kiểm. Không gửi Zalo.
- **Rủi ro kiểm thử trên production** (dev SQLite hay treo): đề xuất dựng staging (một NocoBase + PostgreSQL thứ hai). Đề xuất này chờ chủ dự án duyệt vì liên quan RAM của VPS. Trước khi có staging: kiểm tích hợp trên production chỉ được làm sau khi sao lưu, với dữ liệu `[TEST]`, và dọn ngay sau đó.

## 3. 4.1 Giải đấu — thiết kế chi tiết

### 3.1 Giả định mặc định (chủ dự án đổi được sau)

| #   | Điểm       | Mặc định                                                                                                                                                                                                                                                                                                                                              |
| --- | ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| T1  | Thể thức   | Hệ **Thuỵ Sĩ** (mặc định). Bảng ≤ 8 kỳ thủ được chọn **vòng tròn** (bảng Berger)                                                                                                                                                                                                                                                                      |
| T2  | Xếp cặp    | Bộ xếp cặp Thuỵ Sĩ **tự viết, giản lược theo tinh thần FIDE Dutch**: nhóm điểm, nửa trên gặp nửa dưới, đẩy xuống nhóm dưới, không gặp lại, cân màu, bye cho người thấp nhất chưa từng được bye (bye = 1 điểm). Có **xuất TRF16** để đối chiếu hoặc chạy bằng Swiss-Manager cho giải chính thức. Trọng tài được **đổi cặp tay** trước khi công bố vòng |
| T3  | Hệ số phụ  | Buchholz cắt 1, Buchholz, Sonneborn-Berger, đối đầu trực tiếp, số ván thắng                                                                                                                                                                                                                                                                           |
| T4  | Elo nội bộ | Chỉ tính cho học viên Dương Sinh. Giá trị khởi điểm theo cấp: Tốt 800, Mã 900, Tượng 1000, Xe 1100, Hậu 1200, Vua 1300; chưa có cấp thì 800. K = 40. Công thức kỳ vọng chuẩn Elo. Tính một lần khi **kết thúc giải**, từng ván có ghi lịch sử. Kỳ thủ ngoài chỉ dùng Elo khai báo để xếp hạt giống, không lưu                                         |
| T5  | Đăng ký    | (a) Nhân viên đăng ký học viên CLB, có thể chọn cả lớp một lần. (b) **Trang đăng ký công khai do plugin phục vụ** cho người ngoài: sau khi gửi, hiện ngay mã đăng ký `GD#####` và mã VietQR                                                                                                                                                           |
| T6  | Lệ phí     | Thu theo bảng đấu. Nội dung chuyển khoản `DSC GD#####`. Webhook SePay mở rộng để nhận mã `GD`, tự tạo phiếu thu `loai_thu = le_phi_giai` và xác nhận đăng ký. Học viên CLB được miễn hoặc giảm tuỳ cấu hình của giải                                                                                                                                  |
| T7  | Công bố    | Trang công khai: cặp đấu từng vòng và bảng xếp hạng. **Chỉ hiện họ tên khi có đồng ý công bố** (`dong_y_cong_bo`). Không có đồng ý thì hiện "Kỳ thủ #<số xuất phát>". Chỉ hiện năm sinh, CLB, điểm; không SĐT, không ngày sinh đầy đủ                                                                                                                 |
| T8  | Giấy khen  | Hạng 1–3 mỗi bảng (cấu hình được) in giấy khen A4 ngang, dùng lại khung chứng nhận của 3.3, số `GK#####`                                                                                                                                                                                                                                              |

### 3.2 Dữ liệu (`setup/schema/phase4.mjs`, gọi từ `run.mjs collections`)

| Collection        | Trường chính                                                                                                                                                                                                                                                                                                                                                        | Quan hệ                                                                            |
| ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `dsc_giai_dau`    | `ma` (sequence `GIAI`), `ten`, `slug` (unique, dùng cho trang công khai), `loai` (`noi_bo` / `mo_rong`), `ngay_bat_dau`, `ngay_ket_thuc`, `dia_diem`, `han_dang_ky`, `mo_ta` (markdown), `dieu_le` (markdown), `trang_thai` (`nhap` / `mo_dang_ky` / `dong_dang_ky` / `dang_dien_ra` / `ket_thuc`), `mien_phi_hoc_vien` (checkbox), `so_giai_thuong` (mặc định 3)   | m2o `co_so`                                                                        |
| `dsc_bang_dau`    | `ten` (vd "U8", "Mở rộng"), `the_thuc` (`thuy_si` / `vong_tron`), `so_vong`, `thoi_gian` (vd "15+5"), `le_phi`, `nam_sinh_tu`, `nam_sinh_den`, `vong_hien_tai`, `trang_thai`                                                                                                                                                                                        | m2o `giai_dau`                                                                     |
| `dsc_ky_thu`      | `ma_dang_ky` (sequence `GD`), `ho_ten`, `nam_sinh`, `gioi_tinh`, `clb`, `sdt_lien_he`, `email`, `elo_khai_bao`, `elo_xep`, `so_xuat_phat`, `trang_thai` (`cho_thanh_toan` / `da_xac_nhan` / `rut_lui`), `le_phi`, `da_thu`, `dong_y_du_lieu`, `dong_y_cong_bo`, `diem`, `buchholz_cat1`, `buchholz`, `sonneborn`, `so_thang`, `hang`, `nguon` (`nhan_vien` / `web`) | m2o `bang_dau`, m2o `hoc_vien` (rỗng nếu là người ngoài), m2o `phieu_thu`          |
| `dsc_van_dau`     | `vong`, `ban`, `ket_qua` (`1-0` / `0-1` / `1/2` / `+-` / `-+` / `--` / `bye` / `chua_co`), `pgn` (text, tuỳ chọn), `doi_tay` (checkbox: trọng tài sửa cặp)                                                                                                                                                                                                          | m2o `bang_dau`, m2o `trang` (`dsc_ky_thu`), m2o `den` (`dsc_ky_thu`, rỗng khi bye) |
| `dsc_elo_lich_su` | `elo_truoc`, `elo_sau`, `thay_doi`, `so_van`, `ngay`                                                                                                                                                                                                                                                                                                                | m2o `hoc_vien`, m2o `giai_dau`                                                     |
| `dsc_giay_khen`   | `so` (sequence `GK`), `hang`, `ho_ten_tren_giay`, `ngay_cap`                                                                                                                                                                                                                                                                                                        | m2o `ky_thu`, m2o `bang_dau`, m2o `hoc_vien`                                       |

- Bảng có sẵn được thêm trường:
  - `dsc_hoc_vien.elo` (integer, tự tính), `dsc_hoc_vien.so_van_elo` (integer, tự tính);
  - `dsc_phieu_thu.loai_thu` thêm option `le_phi_giai`, cùng m2o `ky_thu`.
- Khoá duy nhất `(bang_dau, hoc_vien)` khi có `hoc_vien`, kiểm ở plugin. `(bang_dau, vong, ban)` là duy nhất.

**Phân quyền:**

- **Quản lý:** toàn quyền.
- **Giáo viên** (làm trọng tài): sửa được `dsc_ky_thu` và `dsc_van_dau`. Chỉ xem `dsc_giai_dau`, `dsc_bang_dau`, `dsc_elo_lich_su`, `dsc_giay_khen`.
- **Kế toán:** xem mọi bảng, sửa trạng thái thanh toán của kỳ thủ.
- **Tư vấn:** xem.
- **Phụ huynh:** chỉ xem `dsc_ky_thu`, `dsc_van_dau` (các ván con mình đã chơi), `dsc_elo_lich_su`, `dsc_giay_khen` **của con mình**. Cột tường minh, không SĐT hay email của người khác.
  - Tên đối thủ chỉ hiện qua action `standings` công khai, đã áp quy tắc T7. **Không** đi qua appends của `dsc_ky_thu`.
- Thêm các bảng mới vào `parent-isolation.mjs`, `parent-oracle.mjs`, `staff-roles-check.mjs`.

### 3.3 Plugin mới `@duongsinh/plugin-dsc-tournament`

Chép khung từ `plugin-dsc-chess`. Có `client.js` / `server.js` ở gốc.

**Lõi thuần** (thư mục `src/server/core/`, không phụ thuộc NocoBase, test kỹ):

- `swiss.ts`: `pairRound(players, history, options)`, trả về danh sách cặp và bye.
  - Bất biến phải kiểm bằng test, gồm cả kiểm thử ngẫu nhiên nhiều giải giả lập:
    - mỗi kỳ thủ còn thi đấu xuất hiện đúng một lần mỗi vòng;
    - không gặp lại khi còn cách khác (nếu buộc phải gặp lại thì báo cảnh báo);
    - không ai nhận bye hai lần khi còn người chưa bye;
    - không ai cầm cùng một màu 3 ván liên tiếp, và chênh lệch màu ≤ 2 khi còn cách khác;
    - vòng 1 xếp theo hạt giống (`elo_xep` rồi theo tên).
- `roundRobin.ts`: bảng Berger cho 3–16 kỳ thủ. Số lẻ thì thêm bye.
- `standings.ts`: điểm, Buchholz cắt 1, Buchholz, Sonneborn-Berger, đối đầu trực tiếp, số ván thắng. Xử lý thắng/thua do vắng mặt và bye theo quy ước FIDE (bye tính như hoà khi tính Buchholz của đối thủ, theo quy tắc ván ảo; mức giản lược này được ghi rõ trong README).
- `elo.ts`: tính Elo theo T4, dùng chung cho mọi ván của giải.
- `trf.ts`: xuất TRF16.

**Các action** (kiểm quyền bên trong):

- `pair({ bangDauId, vong })` (quản lý, giáo viên): yêu cầu vòng trước đã nhập đủ kết quả. Tạo `dsc_van_dau` trong transaction. Chạy lại cùng vòng thì 409, trừ khi `force` và vòng chưa có kết quả nào.
- `swap({ vanDauId, ... })`: trọng tài đổi cặp tay, có kiểm hợp lệ.
- `result({ vanDauId, ketQua })`: tính lại bảng xếp hạng.
- `standings({ slug | bangDauId })`: **công khai** (ACL `public`), áp quy tắc T7, chỉ cho giải đã `dang_dien_ra` / `ket_thuc`.
- `pairings({ slug, vong })`: công khai, cùng quy tắc.
- `finish({ giaiDauId })` (quản lý):
  - kiểm mọi ván đã có kết quả;
  - tính hạng cuối, Elo (ghi `dsc_elo_lich_su`, cập nhật `dsc_hoc_vien.elo`), và giấy khen hạng 1–N;
  - chạy lại nhiều lần cho cùng kết quả.
- `trf({ bangDauId })`: tải file.
- `certificate({ giayKhenId })`: tương tự chứng nhận ở 3.3. Phụ huynh chỉ lấy được của con mình.

**Đăng ký công khai** (`src/server/public.ts`):

- `GET /api/dscTournament:page?slug=` trả trang HTML nhẹ, tự chứa (không cần client NocoBase): thông tin giải, điều lệ, các bảng, form đăng ký, liên kết tới cặp đấu và bảng xếp hạng.
- `POST /api/dscTournament:register`:
  - kiểm `trang_thai = mo_dang_ky` và hạn đăng ký;
  - kiểm năm sinh thuộc khoảng của bảng;
  - bắt buộc `dong_y_du_lieu`;
  - chống spam: trường bẫy (honeypot), giới hạn tần suất theo IP (ví dụ 5 lần / 10 phút, lưu trong bộ nhớ), giới hạn độ dài các trường.
  - Thành công: tạo `dsc_ky_thu` `cho_thanh_toan`, `nguon = web`, rồi trả trang hiện mã `GD#####`, số tiền và **ảnh VietQR** (cùng `BANK` với `setup/ui/vietqr.mjs`, nội dung `DSC GD#####`). Lệ phí bằng 0 thì đặt `da_xac_nhan` luôn.
- `GET /api/dscTournament:status?ma=GD#####&sdt=...`: tra cứu trạng thái. Phải khớp **cả mã lẫn SĐT**, có giới hạn tần suất. Chỉ trả trạng thái và số tiền.

**Mở rộng `dsc-bank-webhook`:**

- regex tách thêm `GD\s?(\d{5})`;
- ghép vào `dsc_ky_thu`, tạo phiếu thu `loai_thu = le_phi_giai`, gắn `ky_thu`;
- đủ lệ phí thì chuyển `da_xac_nhan`, thiếu thì vẫn ghi nhận và đặt `ly_do`.

Nội dung chuyển khoản có cả mã `DK` lẫn `GD` thì cho vào `cho_xu_ly`.

Thêm unit test và mục kiểm SePay tương ứng.

### 3.4 Workflow

- **W18:** kỳ thủ đăng ký từ web thì báo quản lý và tư vấn trong app.
- **W19:** kỳ thủ chuyển `da_xac_nhan` thì thông báo trong app cho phụ huynh, nếu kỳ thủ là học viên có tài khoản phụ huynh. Hàng đợi Zalo thêm loại `giai_dau` (chỉ gửi khi có mẫu ZNS và Zalo được bật).
- `finish` tạo giấy khen thì thông báo phụ huynh trong app.

### 3.5 Giao diện

- **Desktop**, nhóm mới "Giải đấu":
  - **Giải đấu:** popup có các bảng, nút "Mở đăng ký", liên kết trang công khai.
  - **Kỳ thủ:** lọc theo bảng / trạng thái, nút "Thêm học viên CLB" hoặc "Thêm cả lớp" bằng khối JS gọi action `enroll`.
  - **Vòng đấu:** khối JS chọn bảng và vòng, nút "Xếp cặp vòng N", bảng cặp đấu, nhập kết quả nhanh, nút "Xuất TRF".
  - **Bảng xếp hạng:** khối gọi `standings`.
  - **Giấy khen:** nút In.
- **Mobile giáo viên**, trang "Trọng tài":
  - chọn giải đang diễn ra, rồi chọn bảng, rồi chọn vòng;
  - danh sách bàn đấu, mỗi bàn bấm một chạm để chọn kết quả (1-0 / ½ / 0-1 / vắng);
  - nút "Xếp cặp vòng tiếp theo".

  Giáo viên đang có 5 tab, thêm trang này là 6 (≤ 6). Nếu cần chỗ, gộp "Phòng thi" và "Trọng tài" thành "Thi & giải".

- **Mobile phụ huynh**, trong "Con tôi" thêm mục "Giải đấu & Elo": các giải đã và đang tham gia, ván đấu của con, Elo hiện tại, lịch sử Elo (đường SVG tự vẽ, dùng lại cách vẽ trong `profile.mjs`), giấy khen kèm nút Xem / In. **Không thêm tab.**
- Khối "Tiến bộ" (3.4) thêm Elo nếu có.

## 4. Thứ tự triển khai 4.1

1. **4.1-1** Schema + ACL + `gen-parent-columns` + bộ dò (isolation, oracle, staff) cho 6 bảng mới và các trường mới.
2. **4.1-2** Plugin `dsc-tournament`:
   - lõi `swiss`, `roundRobin`, `standings`, `elo`, `trf`;
   - unit test cùng kiểm thử ngẫu nhiên (ví dụ 200 giải giả lập, 7–60 kỳ thủ, 5–9 vòng, kết quả ngẫu nhiên, kiểm mọi bất biến);
   - các action nội bộ `pair`, `swap`, `result`, `standings`, `pairings`, `finish`, `trf`, `enroll`, `certificate`.
3. **4.1-3** Đăng ký công khai, tra cứu trạng thái, mở rộng webhook SePay mã `GD`, phiếu thu `le_phi_giai`, W18, W19.
4. **4.1-4** Giao diện: desktop "Giải đấu", mobile "Trọng tài", phụ huynh "Giải đấu & Elo", in giấy khen.
5. **Kiểm chứng** mục 5. Chụp ảnh. Ghi mục "Kết quả" cuối file này. Cập nhật Tiến độ của kế hoạch chiến lược, README và file bàn giao.

## 5. Kiểm chứng (`verify.mjs` mục 14, `setup/tournament-check.mjs`)

1. **Giải giả lập 12 kỳ thủ, 5 vòng Thuỵ Sĩ** (`[TEST]`):
   - mỗi vòng đủ cặp, không gặp lại, bye đúng quy tắc;
   - nhập kết quả, bảng xếp hạng đúng theo tính tay của một tình huống cố định (fixture);
   - TRF xuất ra đúng định dạng: kiểm từng dòng `001` và số vòng.
2. **Vòng tròn 5 kỳ thủ:** đủ C(5,2) ván, có bye mỗi vòng.
3. `pair` khi vòng trước còn thiếu kết quả: 409. Giáo viên xếp cặp được, kế toán không (403).
4. **Đăng ký công khai:**
   - hợp lệ thì có mã `GD` và ảnh QR trong HTML;
   - năm sinh ngoài khoảng, đã quá hạn, hay thiếu đồng ý thì 400;
   - honeypot có giá trị thì bị từ chối êm (trả 200 nhưng không tạo bản ghi);
   - vượt giới hạn tần suất thì 429;
   - `status` sai SĐT thì 404.
5. **Webhook mẫu `DSC GD#####`:** tạo phiếu thu `le_phi_giai`, kỳ thủ chuyển `da_xac_nhan`, gửi lại cùng mã giao dịch không tạo trùng. Payload có cả `DK` và `GD` thì `cho_xu_ly`. Các ca webhook `DK` cũ vẫn đạt.
6. **`finish`:** Elo của học viên khớp tính tay cho fixture. Có `dsc_elo_lich_su`. Giấy khen hạng 1–3 (số `GK`). Chạy lại không tạo trùng.
7. **Công khai `standings` / `pairings`:** kỳ thủ không có đồng ý công bố hiện "Kỳ thủ #n". Không trả SĐT, email, ngày sinh. Giải `nhap` thì 404.
8. **Cách ly:** phụ huynh A không xem được kỳ thủ, ván, Elo, giấy khen của con phụ huynh B qua mọi đường (list, get, appends, route, lọc, sắp xếp, action `certificate`).
9. **Nhân viên:** giáo viên nhập kết quả được nhưng không `finish` được; quản lý `finish` được.
10. **Trình duyệt** (Playwright, desktop + iPhone 13, tài khoản thử):
    - quản lý tạo giải, mở đăng ký;
    - đăng ký công khai, thấy mã QR;
    - giáo viên xếp cặp, nhập kết quả trên điện thoại;
    - xem bảng xếp hạng công khai;
    - `finish`, in giấy khen;
    - phụ huynh thử thấy Elo và giấy khen.

    0 lỗi JS.

Sau khi kiểm: dọn `[TEST]`, đặt lại các bộ đếm `GIAI`, `GD`, `GK` và các số phiếu thu đã dùng nếu được.

## 6. Rủi ro

- **Xếp cặp sai làm hỏng giải thật:** có kiểm thử ngẫu nhiên các bất biến, trọng tài đổi cặp tay được, xuất TRF để đối chiếu Swiss-Manager. Với DSC Open (giải chính thức), khuyến nghị chạy song song Swiss-Manager ở giải đầu tiên.
- **Lộ dữ liệu trẻ em trên trang công khai:** quy tắc T7, chỉ trả các trường tối thiểu, có kiểm thử ở mục 5.7. Trang công khai không chứa dữ liệu nhạy cảm.
- **Spam đăng ký công khai:** honeypot, giới hạn tần suất, bắt buộc đồng ý, quản lý được báo để xem lại.
- **Nhầm mã lệ phí với mã học phí:** hai tiền tố riêng (`DK` / `GD`); có cả hai mã thì chuyển xử lý tay.
- **Tải VPS:** xếp cặp chạy trong bộ nhớ, giải 100 kỳ thủ vẫn nhẹ. Trang công khai trả JSON/HTML nhẹ.

## 7. Việc của chủ dự án (khi dùng thật)

- Xác nhận hoặc đổi các giả định T1–T8 (thể thức, Elo khởi điểm, miễn lệ phí cho học viên, quy tắc công bố tên).
- Duyệt đề xuất **staging** (RAM VPS).
- Cung cấp điều lệ mẫu của giải nội bộ hằng quý và DSC Open.
- Tạo mẫu ZNS "đăng ký giải" nếu muốn gửi qua Zalo.

## 8. Kết quả (02/11/2026)

### 4.1-1 + 4.1-2 (schema, ACL, plugin `@duongsinh/plugin-dsc-tournament` 0.1.1) — xong, production

- **Schema** `setup/schema/phase4.mjs`: 6 bảng (`dsc_giai_dau`, `dsc_bang_dau`, `dsc_ky_thu`, `dsc_van_dau`, `dsc_elo_lich_su`, `dsc_giay_khen`; sequence `GIAI` / `GD` / `GK`). Bảng cũ: `dsc_hoc_vien.elo` + `so_van_elo`, `dsc_phieu_thu.loai_thu = le_phi_giai` + `ky_thu`.
- **ACL** (`setup/acl/apply.mjs`): quản lý toàn quyền; giáo viên sửa `dsc_ky_thu` / `dsc_van_dau`, chỉ xem phần còn lại; kế toán xem + sửa tiền / trạng thái kỳ thủ (`fields` giới hạn); tư vấn xem; phụ huynh chỉ của CON MÌNH, cột tường minh (không SĐT / email / đồng ý công bố; không có quan hệ `trang` / `den` của ván đấu nên không lộ tên đối thủ qua appends). `gen-parent-columns` đã sinh lại, guard 0.1.7.
- **Lõi thuần** `src/server/core/`: `swiss.ts` (Dutch giản lược, nới dần: bye → gặp lại → màu, mỗi lần nới có cảnh báo), `roundRobin.ts` (xoay vòng, cân bằng màu), `standings.ts` (điểm, Buchholz cắt 1, Buchholz, Sonneborn-Berger, đối đầu, số thắng, hạng), `elo.ts`, `trf.ts` (TRF16). Quy ước giản lược (Buchholz với ván không đánh trên bàn dùng đối thủ ảo kiểu FIDE C.07; SB chỉ tính ván trên bàn; số thắng không tính bye / thắng do vắng) ghi trong đầu file.
- **Action**: `enroll`, `pair` (409 khi vòng trước thiếu kết quả / chạy lại, `force` chỉ khi chưa có kết quả), `swap`, `result`, `finish` (chỉ quản lý, chạy lại không đổi), `trf`, `certificate`; công khai `standings`, `pairings` (quy tắc T7, giới hạn 60 yêu cầu / phút / IP). Thêm `refereeStandings` / `refereePairings` cho trọng tài đăng nhập: action `public` của NocoBase KHÔNG nhận diện người đăng nhập (`currentUser` rỗng), nên bản công khai luôn áp quy tắc công khai; giao diện nội bộ dùng hai action này.
- **Kiểm chứng**: lõi 10 test (fixture tính tay: standings 4 kỳ thủ, Elo 1000 thắng 1200 → +30 / −30, TRF từng cột; **240 giải ngẫu nhiên, 1667 vòng**, 7–60 kỳ thủ, có rút lui và ván bỏ cuộc: mỗi người đúng một lần, không bye lần hai, gặp lại chỉ khi không còn cách khác — đối chiếu quy hoạch bitmask ≤ 22 người —, mọi vi phạm màu đều có cảnh báo; kết quả: 10 vòng buộc gặp lại, 2 vòng vi phạm màu, bye thấp nhất 793/794) + 9 test nghiệp vụ (Store giả). `verify.mjs` mục 14 (`setup/tournament-check.mjs`, 23 phép: Thuỵ Sĩ 12 × 5, vòng tròn 5, bye, công khai, finish, Elo, giấy khen, cách ly). Bộ dò mới: `parent-isolation.mjs` 7m, `parent-oracle.mjs` (11 phép chặn + 3 phép hợp lệ), `staff-roles-check.mjs`.

### Lệch kế hoạch

- Bản dev SQLite không chạy tích hợp (treo ở các phép nặng): kiểm tích hợp chạy thẳng trên production PostgreSQL sau khi sao lưu, đúng ràng buộc mục 2.
- Lõi import tương đối kèm đuôi `.ts` để Node ≥ 22.18 chạy test; `plugins/build.mjs` tự bỏ đuôi này khi build.
- Chưa làm (task sau): đăng ký công khai + QR, webhook `GD`, W18 / W19, giao diện.

### 4.1-3 Đăng ký công khai, VietQR, webhook mã `GD`, W18 / W19 (02/11/2026) — xong, dev + production

- **Plugin `dsc-tournament` 0.2.2** (`src/server/public.ts`, `service.ts`):
  - `GET /api/dscTournament:page?slug=` → HTML tự chứa (thông tin giải, điều lệ markdown, bảng đấu, form có honeypot, liên kết cặp đấu / xếp hạng / tra cứu qua `&xem=`). Mọi dữ liệu động đi qua `esc()`; markdown thoát trước rồi chỉ bật tiêu đề, danh sách, đậm / nghiêng, liên kết http(s); CSP `default-src 'none'; img-src https://img.vietqr.io; form-action 'self'` (không script, không cookie nên không cần CSRF).
  - `POST /api/dscTournament:register` (JSON hoặc form-urlencoded, kiểu khác → 415): kiểm giải `mo_dang_ky`, hạn, năm sinh trong khoảng bảng, `dong_y_du_lieu`, độ dài / ký tự điều khiển, trùng họ tên + SĐT (409); honeypot → luôn 200, không tạo, không qua giới hạn (không đưa tín hiệu 429 cho bot); giới hạn 5 lượt / 10 phút / IP chỉ tính đăng ký sẽ TẠO (guard chạy sau khi dữ liệu hợp lệ, trước khi tạo) → 429; lệ phí 0 → `da_xac_nhan`; trang kết quả có mã `GD#####`, số tiền, ảnh VietQR (cùng `BANK` với `setup/ui/vietqr.mjs`, có test đối chiếu).
  - `GET /api/dscTournament:status?ma=&sdt=`: phải khớp cả hai, sai một trong hai cùng một thông báo 404 (không dò được mã), 10 lượt / phút, chỉ trả trạng thái và số tiền. IP người gọi lấy phần tử cuối của `X-Forwarded-For` khi kết nối trực tiếp là địa chỉ nội bộ (Caddy ghi, người dùng không giả được).
- **`dsc-bank-webhook` 0.2.0**: `extractTournamentCodes` (`GD\s?\d{5}`); nội dung có cả DK và GD, hoặc nhiều GD → `cho_xu_ly`; một mã GD → phiếu thu `le_phi_giai` gắn `ky_thu`, cộng `da_thu`, đủ lệ phí thì `da_xac_nhan` (thiếu / thừa vẫn ghi nhận, `ly_do` ghi chênh lệch), mã lạ / kỳ thủ rút lui → `cho_xu_ly`; gửi lại cùng id không tạo trùng. Công thức học phí (F3) và ZNS W10 chỉ xét `loai_thu = hoc_phi` nên không bị ảnh hưởng. Ghi chú: phiếu `le_phi_giai` kế toán nhập tay không tự cập nhật `da_thu` của kỳ thủ; kế toán sửa `da_thu` / `trang_thai` của kỳ thủ (ACL cho phép).
- **W18** (đăng ký web → báo quản lý + tư vấn), **W19** (`da_xac_nhan` → báo phụ huynh trong app nếu là học viên có tài khoản; Zalo `giai_dau` chỉ khi cấu hình có `ma_mau_giai_dau`, `dsc-zalo` 0.1.3, vẫn chế độ `tat`).
- **Kiểm chứng**: 118 test đơn vị (webhook +5, công khai 10); `verify sepay` mục 11 (11.12 mới) đạt TRÊN DEV bằng khoá thử (54 phép, kể cả mọi ca DK cũ), production chỉ 503 / bỏ qua gọi webhook; `verify tournament` mục 14.6–14.10; ảnh `reports/` + `reports/production/giai-dau-dang-ky-{desktop,iphone}-*.png`.
- **Lần này dev chạy được** (khởi động ~10 phút, cài plugin tự khởi động lại ~5 phút): cài 4 plugin, áp schema + workflow, chạy `verify sepay` đầy đủ, `verify tournament` và Playwright. Dev SQLite thỉnh thoảng trả 500 (SQLITE_BUSY) ở ghi đồng thời nên các phép kiểm thử lại có thử lại.

### 4.1-4 Giao diện giải đấu + `da_thu` kỳ thủ tự tính (02/11/2026) — xong

- **`dsc_ky_thu.da_thu` là trường tự tính duy nhất** = Σ phiếu thu `da_thu`, `loai_thu = le_phi_giai`, gắn `ky_thu` (`formulas.mjs`: `playerPaid` / `playerStatus`, có trong FN / FM và `recalc.mjs`). Workflow **F10** (phiếu thu tạo / sửa / xoá) tính lại đã thu rồi đổi trạng thái `cho_thanh_toan` ↔ `da_xac_nhan` theo `da_thu >= le_phi` bằng node `update` (để W19 phát đúng một lần), không đụng `rut_lui`, lệ phí 0 → `da_xac_nhan`; **F11** khi đổi `le_phi`. Webhook `dsc-bank-webhook` 0.2.1 KHÔNG cộng `da_thu` nữa (chỉ tạo phiếu thu) nên webhook + phiếu tay không cộng đôi. ACL: kế toán chỉ sửa `le_phi` của kỳ thủ; trọng tài sửa thông tin kỳ thủ nhưng không sửa trường tự tính. Kế toán nhập phiếu ở Học phí › Phiếu thu (loại "Lệ phí giải đấu" + chọn kỳ thủ).
- **Server (`dsc-tournament` 0.3.0)**: `mySummary` (Elo hiện tại + lịch sử, giải đã / đang tham gia kèm hạng, ván của con với tên đối thủ theo T7, giấy khen; phụ huynh chỉ con mình), `refereeOverview` (giải + bảng + lớp), `findStudents`.
- **Desktop nhóm "Giải đấu"** (quản lý + giáo viên): Giải đấu (Mở / Đóng đăng ký, liên kết công khai, bảng liên quan), Bảng đấu, Kỳ thủ (tab "Thêm học viên CLB / cả lớp" gọi `enroll`), Vòng đấu (chọn giải → bảng → vòng; Xếp cặp vòng N, nhập kết quả một chạm, Đổi màu, Xuất TRF, Kết thúc giải bấm hai lần), Bảng xếp hạng, Giấy khen (xem trước + In A4 ngang, khung chứng nhận 3.3).
- **Mobile giáo viên**: tab thứ 6 "Trọng tài" (6 tab). **Phụ huynh**: mục "Giải đấu & Elo" (Elo + đường lịch sử SVG, giải, ván của con, giấy khen Xem / In) nằm TRONG khối "Tiến bộ" ở "Con tôi" — validator của flowSurfaces từ chối popup có hơn 3 khối cạnh chi tiết học viên, nên không tách thành khối riêng; vẫn không thêm tab. Cùng khối cũng hiện ở "Tra cứu học viên" của giáo viên.
- **Kiểm chứng**: 122 test đơn vị (công thức SQLite cho `da_thu` / trạng thái, `mySummary`, UI); Playwright kịch bản 5.10 (`setup/ui-shots-giai-dau.mjs`) đạt 29 phép trên dev VÀ trên production, 0 lỗi JS; `verify` đầy đủ production 592 phép đạt, `verify sepay` (dev, khoá thử) 59 phép đạt. Sửa trong lúc kiểm: F10 lọc "trạng thái chưa là đích" để hai lần chạy chồng không phát W19 hai lần; khối Vòng đấu / Trọng tài bỏ ván của vòng cũ ngay khi đổi vòng (trước đó thẻ ván cũ hiện tạm dưới nhãn vòng mới); `da_thu` / `le_phi` mặc định 0.
