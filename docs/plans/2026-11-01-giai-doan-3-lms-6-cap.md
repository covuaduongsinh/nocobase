# Kế hoạch giai đoạn 3 — LMS 6 cấp: giáo trình số, bàn cờ tương tác, bài tập về nhà

Thuộc kế hoạch chiến lược `docs/plans/2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md` (giai đoạn 3: "Edu tech: LMS 6 cấp"). Lập ngày 01/11/2026.

## 1. Bối cảnh và rà soát

**Tiến độ đến 01/11/2026:**

- Giai đoạn 1 và bộ máy tính toán đã xong.
- Giai đoạn 2 đã xong phần mã của các bước 2.1–2.5:
  - Zalo ZNS (`f69b6d5`…`0b7fcb0`) đang chờ chủ dự án gửi App ID/secret, 3 mẫu ZNS và refresh token.
  - SePay (`b701122`, `953617d`) đang chờ chủ dự án gửi khoá.
- Bước 2.6 (kích hoạt phụ huynh hàng loạt) chưa làm, vì còn phải bổ sung dữ liệu phụ huynh. Chủ dự án chọn chuyển sang giai đoạn 3.

**Giai đoạn 3 trong kế hoạch chiến lược gồm 7 mảng:**

- 3.1 giáo trình số;
- 3.2 bài tập về nhà trên bàn cờ;
- 3.3 thi lên cấp và chứng nhận;
- 3.4 hồ sơ năng lực theo thời gian;
- 3.5 hợp nhất app học cờ / thuviencovua.com;
- 3.6 trình học cho trẻ mầm non (`kids-player`);
- 3.7 khoá học online trả phí (freemium).

**Chủ dự án chốt ngày 01/11/2026:**

1. Lượt này làm **3.1 giáo trình + bàn cờ** và **3.2 bài tập về nhà tự chấm**. Đây là nền cho 3.3–3.7, các mảng đó sẽ có kế hoạch riêng sau.
2. Học viên làm bài **bằng tài khoản phụ huynh** (chọn tên con khi làm). Không cấp tài khoản cho trẻ em.
3. Học liệu **chưa có, soạn mới**. Hệ thống dựng khung, công cụ nhập và vài bài mẫu; giáo viên soạn dần.

**Thước đo:** % học viên đang học làm ít nhất 1 bài tập online mỗi tuần. Số bài học và thế cờ đã xuất bản theo từng cấp.

## 2. Quyết định kỹ thuật

1. **Bàn cờ dùng `jsBlock` + `ctx.importAsync`, không viết plugin client.**
   - Đây là cùng cơ chế đã dùng cho khối VietQR (`setup/ui/vietqr.mjs`).
   - RunJS của NocoBase 2.2.36 nạp được module ESM theo URL (`packages/core/flow-engine/src/utils/runjsModuleLoader.ts`, CDN mặc định là `https://esm.sh`, đổi được qua `window.__esm_cdn_base_url__`).
   - Lý do: máy dev không đủ RAM để chạy client dev, và build plugin client v2 là rủi ro lớn. Mã khối được sinh từ script và lưu trong repo, nên vẫn tái chạy an toàn.
   - **Thư viện:** `chess.js` (BSD-2, luật cờ, FEN/PGN) và một bàn cờ **giấy phép MIT**: ưu tiên `cm-chessboard`, phương án dự phòng là `react-chessboard`. **Không dùng `chessground`** (GPL-3). **Ghim phiên bản** trong URL.
   - **Nguồn tải:** bước 1 dùng esm.sh. Nếu thử nghiệm cho thấy esm.sh chậm hoặc chập chờn ở Việt Nam, plugin `dsc-chess` sẽ tự phục vụ file JS đã đóng gói. Bước 3.1-0 phải quyết định điểm này.
2. **Chấm bài ở server, không tin trình duyệt.**
   - Plugin server-only `@duongsinh/plugin-dsc-chess` có action `dscChess:submit`.
   - Action dùng `chess.js` để kiểm lại toàn bộ chuỗi nước đi, ghi kết quả và cộng sao.
   - Lời giải **không bao giờ gửi xuống tài khoản phụ huynh**: ACL bỏ cột `loi_giai`, và guard chặn cả appends.
3. **Bàn cờ cần nước đáp của máy để chạy bài nhiều nước.**
   - Cách làm: client gửi từng nước lên `dscChess:move`.
   - Server trả về đúng/sai và nước đáp kế tiếp. Không lộ phần lời giải còn lại.
   - Server giữ trạng thái lần làm trong `dsc_lam_bai`.
4. **Cộng sao** bằng cách tạo `dsc_giao_dich_sao` với `loai = 'bai_tap'` (thêm option này vào select). F5 sẵn có sẽ tự tính lại tổng sao. Mỗi thế cờ chỉ cộng sao một lần cho mỗi học viên.

## 3. Mô hình dữ liệu (file mới `setup/schema/phase3.mjs`, gọi từ `run.mjs collections`)

| Collection           | Trường chính                                                                                                                                                                                                                                                                                                                                                                                | Quan hệ                                                                                                             |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `dsc_cap_do` (đã có) | thêm `mo_ta_ngan`, `anh` (tuỳ chọn)                                                                                                                                                                                                                                                                                                                                                         | —                                                                                                                   |
| `dsc_bai_hoc`        | `ma` (sequence `BH`), `ten`, `thu_tu`, `muc_tieu` (text), `noi_dung` (markdown), `video_url`, `trang_sach`, `trang_thai` (`nhap`/`xuat_ban`)                                                                                                                                                                                                                                                | m2o `cap_do`, m2o `sach`                                                                                            |
| `dsc_the_co`         | `ma` (sequence `TC`), `de_bai` (vd "Trắng đi, chiếu hết sau 2 nước"), `fen`, `loi_giai` (chuỗi UCI cách nhau bởi dấu cách, gồm cả nước đáp), `loai` (`chieu_het_1`/`chieu_het_2`/`chieu_het_3`/`chien_thuat`/`tan_cuoc`/`khai_cuoc`/`khac`), `chap_nhan_moi_chieu_het` (checkbox: nước cuối là chiếu hết bất kỳ thì vẫn đúng), `do_kho` (1–5), `so_sao` (mặc định 1), `goi_y`, `trang_thai` | m2o `bai_hoc`, m2o `cap_do`                                                                                         |
| `dsc_giao_bai`       | `ten`, `han_nop` (date), `ghi_chu`, `trang_thai` (`dang_mo`/`da_dong`)                                                                                                                                                                                                                                                                                                                      | m2o `lop`, m2o `giao_vien` (`dsc_nhan_su`), m2m `the_co` (bảng nối `dsc_giao_bai_the_co`), m2o `bai_hoc` (tuỳ chọn) |
| `dsc_lam_bai`        | `buoc` (số nước đúng đã đi), `nuoc_da_di` (text UCI), `ket_qua` (`dang_lam`/`dung`/`sai`/`bo_cuoc`), `so_lan_sai`, `bat_dau_luc`, `xong_luc`, `sao_nhan`                                                                                                                                                                                                                                    | m2o `hoc_vien`, m2o `the_co`, m2o `giao_bai`                                                                        |

- `dsc_lam_bai` có khoá duy nhất `(hoc_vien, the_co, giao_bai)`. Làm lại thì cập nhật cùng dòng (`so_lan_sai` tăng). Bài "luyện tự do" không thuộc giao bài nào thì `giao_bai = NULL`, kiểm tính duy nhất ở plugin.
- **Tự tính (thêm vào `formulas.mjs` và `recalc.mjs`):** `dsc_giao_bai.so_da_xong` / `si_so`, `dsc_hoc_vien.so_bai_tap_tuan` (số thế cờ `dung` trong 7 ngày). FN tính lại hằng đêm.

## 4. Phân quyền (`setup/acl/apply.mjs`)

- `dsc_quan_ly`, `dsc_giao_vien`: được thêm, sửa, xem `dsc_bai_hoc`, `dsc_the_co`, `dsc_giao_bai`; chỉ xem `dsc_lam_bai`.
- `dsc_phu_huynh`:
  - `dsc_bai_hoc` và `dsc_cap_do`: chỉ xem bản ghi `xuat_ban`.
  - `dsc_the_co`: chỉ xem bản ghi `xuat_ban`, **cột tường minh không gồm `loi_giai`**.
  - `dsc_giao_bai`: chỉ xem bài giao cho lớp có con mình, scope `lop.hoc_vien.phu_huynh.tai_khoan.id = $user.id`.
  - `dsc_lam_bai`: chỉ xem của con mình. **Không có quyền ghi**; mọi thao tác ghi đi qua action của plugin.
- Thêm 5 collection vào `setup/parent-isolation.mjs`. Bắt buộc có phép dò "phụ huynh không đọc được `loi_giai`" qua list, get, appends lồng nhau (`giao_bai.the_co`, `bai_hoc.the_co`) và route quan hệ.

## 5. Plugin `@duongsinh/plugin-dsc-chess` (server-only, khung chép từ `plugin-dsc-bank-webhook`)

- Phụ thuộc: `chess.js` (bundle vào plugin, kiểm tra `nocobase-build` có đóng gói dependency không; nếu không thì khai báo trong `dependencies`).
- Các action (đăng ký qua `resourceManager.define({ name: 'dscChess', actions })`, ACL `loggedIn`, tự kiểm quyền bên trong):
  - `start({ theCoId, hocVienId, giaoBaiId? })`:
    - Kiểm tra người gọi:
      - phụ huynh: học viên phải là con của mình qua `dsc_hoc_vien_phu_huynh` + `dsc_phu_huynh.tai_khoan`;
      - giáo viên / quản lý: được làm thử với `hocVienId` rỗng, không ghi.
    - Thế cờ phải `xuat_ban`. Nếu có `giaoBaiId`, bài giao phải mở và thuộc lớp của em.
    - Trả `{ fen, de_bai, ben_di, buoc }`, **không trả lời giải**. Tạo hoặc lấy lại dòng `dsc_lam_bai`.
  - `move({ lamBaiId, uci })`:
    - Kiểm quyền như trên, rồi kiểm nước đi hợp lệ bằng `chess.js`.
    - So với nước kế tiếp trong lời giải. Nếu `chap_nhan_moi_chieu_het` và đây là nước cuối thì chấp nhận mọi nước tạo chiếu hết.
    - Đúng: tăng `buoc`, trả nước đáp của máy (nếu còn) cùng `fen` mới. Xong bài thì `dung`, cộng sao một lần, trả `{ xong: true, sao }`.
    - Sai: `so_lan_sai + 1`, trả `{ dung: false }` và giữ nguyên thế cờ.
  - `hint({ lamBaiId })`: trả `goi_y`, và từ lần sai thứ 2 trở đi thì trả thêm ô xuất phát của nước đúng. Dùng gợi ý thì sao nhận giảm (quy tắc: `so_sao` − 1, tối thiểu 0).
  - `validate({ fen, loi_giai })` (giáo viên / quản lý): kiểm FEN hợp lệ và chuỗi lời giải đi được. Dùng ở nút "Kiểm tra thế cờ" khi soạn bài.
- Mọi ghi chép chạy trong transaction. Cộng sao bằng repository để F5 chạy.
- Unit test (`node:test` hoặc vitest, theo `setup/tests/*.test.mjs`): kiểm lời giải, nước sai, chiếu hết thay thế, nước phong cấp (`e7e8q`), không lộ lời giải, kiểm quyền cha–con, cộng sao đúng một lần.

## 6. Giao diện

**Khối bàn cờ (`setup/ui/chess-board.mjs`, mẫu theo `vietqr.mjs`)** có 3 biến thể dùng chung một lõi mã:

- `xem` (desktop / giáo viên): hiện thế cờ `fen` của bản ghi trong popup, có nút "Kiểm tra thế cờ" gọi `validate`.
- `lam_bai` (mobile phụ huynh):
  - chọn con (nếu nhiều con), gọi `start`;
  - cho kéo hoặc chạm để đi quân, mỗi nước gọi `move`;
  - hoạt hình nước đáp, báo đúng/sai, nút "Gợi ý";
  - màn "Hoàn thành +N ⭐".
  - Trên điện thoại cần ô to, bàn cờ rộng 100%, hỗ trợ chạm (cm-chessboard có hỗ trợ).
- a11y: có nhãn ARIA cho bàn cờ, thông báo kết quả qua `aria-live`, có nút "Đi lại".

**Desktop (`setup/ui/pages.mjs`)** thêm nhóm menu "Học liệu", có các trang:

- Bài học (lọc theo cấp, popup có bảng thế cờ của bài);
- Thế cờ (popup Xem có khối bàn cờ);
- Giao bài (theo lớp, chọn nhiều thế cờ, hạn nộp);
- Kết quả bài tập (lọc lớp / học viên).

Bảng điều khiển thêm biểu đồ "% học viên làm bài trong tuần theo lớp" (SQL tương thích PostgreSQL và SQLite).

**Mobile giáo viên (`setup/ui/mobile-pages.mjs`):**

- Trang "Giao bài": tạo bài giao cho lớp hôm nay.
- Popup lớp hiện ai đã xong / chưa làm.

**Mobile phụ huynh:**

- Trang "Bài tập": bài được giao còn mở của các con, kèm số thế cờ đã xong / tổng.
- Popup bài giao có danh sách thế cờ, mở popup làm bài với khối `lam_bai`.
- Trang "Bài học": bài học đã xuất bản theo cấp hiện tại của con. Nội dung markdown, video nhúng qua URL.
- Tên trang không trùng tên nhóm desktop (bẫy `menu-access.mjs`).
- Quyền menu chỉ cấp cho `dsc_phu_huynh`.

**Bẫy đã biết:** `FIELD_GROUPS` phải phủ mọi trường. Popup có 2 bảng liên quan hoặc khối details có hơn 14 trường sẽ làm validator báo nhầm. Dùng `try-one.mjs` để thử từng trang.

## 7. Học liệu ban đầu và công cụ soạn

- `setup/hoc-lieu.mjs`:
  - `import <file.csv>`: CSV gồm cột `cap, bai_hoc, ma_the_co, de_bai, fen, loi_giai, loai, do_kho, so_sao, goi_y`. Script kiểm từng dòng bằng chính hàm kiểm của plugin (dùng chung module) và báo dòng lỗi. Chạy lại an toàn theo `ma_the_co`.
  - `template`: xuất file CSV mẫu.
- Tạo sẵn **1 bài học mẫu cho mỗi cấp** (khung trống, trạng thái `nhap`).
- Tạo **khoảng 20 thế cờ mẫu** cho cấp Tốt và Mã: chiếu hết 1 nước và 2 nước. Lời giải phải được kiểm bằng `chess.js` trong test. Chỉ dùng thế cờ **tự soạn** hoặc thế cờ kinh điển thuộc phạm vi công cộng; không chép từ sách hay trang có bản quyền. Các mẫu này để trạng thái `nhap` để giáo viên duyệt trước khi xuất bản.
- Viết `docs/huong-dan-soan-the-co.md` cho giáo viên:
  - lấy FEN từ bàn cờ chỉnh sửa của lichess;
  - ghi lời giải dạng UCI;
  - dùng nút "Kiểm tra thế cờ".

## 8. Thứ tự triển khai (mỗi bước: dev → verify → production → commit)

- **3.1-0 Thử khả thi (làm trước, báo lead trước khi đi tiếp):**
  - Dựng một trang thử trên production với dữ liệu `[TEST]`, hoặc trên dev nếu client dev chạy được. Trang có `jsBlock` nạp `chess.js` và `cm-chessboard` qua `ctx.importAsync` (phiên bản ghim), hiện thế cờ, đi được quân trên điện thoại.
  - Đo thời gian tải. Ghi lại bằng ảnh chụp: Playwright nếu có, nếu không thì nhờ chủ dự án mở trên điện thoại.
  - **Không đạt** thì dừng lại và đề xuất phương án: tự phục vụ file JS từ plugin, hoặc đổi thư viện.
- **3.1-1:** schema `phase3.mjs` + ACL + seed các option mới. Chạy `run.mjs collections roles`.
- **3.1-2:** plugin `dsc-chess` + unit test, build và cài trên dev.
- **3.1-3:** giao diện desktop (Học liệu) + khối bàn cờ `xem` + `hoc-lieu.mjs` + học liệu mẫu.
- **3.2-1:** giao diện mobile giáo viên (Giao bài) và phụ huynh (Bài tập, Bài học) + khối `lam_bai`.
- **3.2-2:** công thức tự tính và dashboard. Mở rộng `verify.mjs` thêm mục 12 (xem mục 9).
- **3.2-3:**
  - sao lưu production;
  - cài plugin, chạy `run.mjs collections roles workflows ui`;
  - chạy `verify.mjs` và `parent-isolation.mjs`, cả hai phải "tất cả đạt";
  - nạp học liệu mẫu ở trạng thái `nhap`.
- **Cuối:** commit Conventional Commits (tiếng Anh, `Co-Authored-By`). Cập nhật README, file bàn giao, mục Tiến độ của kế hoạch chiến lược, và thêm mục "Kết quả" vào cuối file này.

## 9. Kiểm chứng (`verify.mjs` mục 12)

1. Phụ huynh A gọi `start` cho con của phụ huynh B: 403. Gọi cho thế cờ `nhap`: 403.
2. Phụ huynh không đọc được `loi_giai` qua bất kỳ đường nào: list, get, appends lồng nhau, route quan hệ, và phản hồi của `start` / `move` / `hint`.
3. Bài chiếu hết 2 nước:
   - đi đúng 2 nước: `dung`, cộng đúng `so_sao` một lần, `tong_sao` của học viên tăng (F5);
   - làm lại: không cộng thêm sao.
4. Nước sai: `so_lan_sai` tăng, thế cờ giữ nguyên. Nước không hợp lệ: 400.
5. `chap_nhan_moi_chieu_het` chấp nhận một nước chiếu hết khác với lời giải.
6. Phong cấp và nhập thành đi đúng.
7. Gợi ý làm giảm sao theo quy tắc.
8. Phụ huynh không tạo, sửa, xoá được `dsc_lam_bai`, `dsc_the_co`, `dsc_giao_bai`.
9. Giáo viên `validate` một FEN sai: báo lỗi rõ ràng. `hoc-lieu.mjs import` báo đúng dòng lỗi.
10. Số liệu tự tính `so_da_xong`, `so_bai_tap_tuan` đúng. `recalc.mjs --check` = 0 lệch.
11. Kiểm tra giao diện trên điện thoại bằng tài khoản phụ huynh **thử** (không đăng nhập thay phụ huynh thật, vì sẽ hỏng số liệu kích hoạt).

## 10. Rủi ro

- **CDN esm.sh ngừng hoặc chậm:** bàn cờ không tải được. Đã có phương án tự phục vụ từ plugin, và bước 3.1-0 kiểm sớm.
- **Lộ lời giải:** đây là rủi ro chính về toàn vẹn bài tập. Chặn bằng ACL, guard và kiểm thử dò mọi đường.
- **Trẻ dùng điện thoại của bố mẹ:** khối làm bài phải đơn giản, chữ to. Không hiện thông tin tài chính trên trang Bài tập.
- **Giáo viên ngại soạn:** cung cấp import CSV, hướng dẫn, mẫu và nút kiểm tra. Đo số thế cờ xuất bản mỗi tuần.
- **Tải VPS:** chấm bài theo từng nước rất nhẹ. Trang bài tập chỉ nạp thế cờ khi mở.
- **Bản quyền học liệu:** chỉ dùng nội dung tự soạn hoặc thuộc phạm vi công cộng.

## 11. Việc của chủ dự án

- Xem thử bàn cờ trên điện thoại ở bước 3.1-0.
- Chỉ định giáo viên soạn học liệu, duyệt và xuất bản thế cờ mẫu.
- Xác nhận quy tắc sao: mặc định 1 sao mỗi thế cờ, dùng gợi ý thì trừ 1.

## 12. Kết quả (01/11/2026)

Đã xong toàn bộ bước 3.1-0 → 3.2-2 (dev + production), commit trong `D:\code\duongsinh-nocobase` (`cae91af`, `6ab215e`, `4d60646`, và 3.2-2).

- **Dữ liệu & quyền:** `dsc_bai_hoc`, `dsc_the_co`, `dsc_giao_bai` (+ bảng nối), `dsc_lam_bai`; phụ huynh chỉ xem bản xuất bản, cột tường minh không có `loi_giai` / `goi_y` / `nuoc_da_di`, không ghi được; **bài học chỉ hiện theo CẤP của con** (data scope `cap_do → học viên → phụ huynh → tài khoản`; bài chưa gắn cấp thì phụ huynh không thấy). Guard 0.1.5 đóng mặc định theo danh sách cột duyệt sinh từ `apply.mjs` (bài học 01/11/2026: plugin-acl tự thêm trường mới vào whitelist mọi vai trò).
- **Plugin `@duongsinh/plugin-dsc-chess` 0.1.2** (chess.js bundle trong plugin): action `start / move / hint / validate / assignment / children / progress`; chấm ở server, không bao giờ trả lời giải; cộng sao một lần qua repository (F5 tự tính lại). Lưu ý: không truyền `transaction` vào `Repository.find/findOne` có `appends` (deep-clone transaction làm văng TypeError).
- **Giao diện:** desktop nhóm "Học liệu" (Bài học, Thế cờ có bàn cờ + "Kiểm tra thế cờ", Giao bài có "Tiến độ lớp", Kết quả bài tập); mobile giáo viên "Giao bài" (+ tiến độ: ai đã xong / đang làm / CHƯA làm); mobile phụ huynh "Bài tập" (bài được giao + luyện thêm, khối làm bài chạy ngay trong popup) và "Bài học" (markdown + video); "Con tôi" hiện số bài tập trong tuần. Thanh tab: phụ huynh 5, giáo viên / quản lý 4 (≤ 6, không phải gộp). Bàn cờ: `jsBlock` + chess.js 1.4.0 (esm.sh) + cm-chessboard 8.15.4 (jsDelivr) ghim phiên bản.
- **Tự tính (3.2-2):** `dsc_giao_bai.si_so` / `so_da_xong` (xong = đúng MỌI thế cờ xuất bản của bài), `dsc_hoc_vien.so_bai_tap_tuan` (thế cờ đúng trong 7 ngày); workflow F8 (lần làm bài), F9 (bài giao), FN / FM; `formulas.test.mjs` đạt trên SQLite và trên PostgreSQL 16 thật (schema tạm + ROLLBACK); `recalc.mjs` đối soát 0 lệch; biểu đồ dashboard "% học viên làm bài trong tuần theo lớp".
- **Học liệu mẫu:** `setup/data/the-co-mau.csv` — 20 thế cờ tự soạn (10 chiếu hết 1 nước cấp Tốt, 10 chiếu hết 2 nước cấp Mã, mỗi thế 2 nước chỉ có đúng 1 nước đầu thắng), 6 bài học khung; tất cả ở trạng thái `nhap` chờ giáo viên duyệt. Công cụ `setup/hoc-lieu.mjs` + `docs/huong-dan-soan-the-co.md`.
- **Kiểm chứng:** `verify.mjs` mục 12 (`setup/lms-check.mjs`, `node setup/verify.mjs lms`) 10 kịch bản API + kịch bản 11 bằng trình duyệt (`setup/ui-shots-mobile.mjs`, iPhone 13, phụ huynh thử, giải thế cờ 2 nước tới "Hoàn thành", sai một nước, gợi ý). Production: `verify.mjs` tất cả đạt (474 phép dò cách ly, 0 rò rỉ).
- **Phát sinh ngoài kế hoạch:** (1) vai trò nhân viên chỉ đọc được cột hệ thống vì `fields = []` → `apply.mjs` cấp `fields` tường minh + `verify.mjs staff`; (2) cửa sổ sai quyền khi chạy `collections` trước `roles` → `collections` luôn kéo theo `roles`, guard fail-closed; (3) giáo viên sửa được mọi thông tin học viên (chủ dự án quyết định).
- **Chưa làm / ghi nhận:** giao diện quản trị chưa có chọn-thế-cờ-từ-ngân-hàng đẹp (chọn nhiều thế cờ bằng ô chọn quan hệ); xếp hạng / huy hiệu / ELO không nằm trong phạm vi; bản dev SQLite chập chờn `SQLITE_BUSY` khi nhiều workflow chạy (verify trên dev cần thử lại, PostgreSQL production ổn).
