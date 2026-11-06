# Kế hoạch giai đoạn 3 (tiếp) — 3.3 Thi lên cấp + chứng nhận, 3.4 Hồ sơ năng lực

Thuộc kế hoạch chiến lược `docs/plans/2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md` (giai đoạn 3) và nối tiếp `docs/plans/2026-11-01-giai-doan-3-lms-6-cap.md` (đã xong 3.1–3.2, xem mục 12 của file đó). Lập ngày 02/11/2026.

## 1. Bối cảnh và rà soát

**Đã xong đến 01/11/2026:**

- Giai đoạn 1.
- Giai đoạn 2, bước 2.1–2.5. Zalo và SePay đang tắt, chờ chủ dự án gửi khoá.
- Giai đoạn 3, bước 3.1–3.2: học liệu, bàn cờ, bài tập về nhà tự chấm, tiến độ lớp. Commit cuối `f6a6004`. Trên production, verify đạt 474 phép dò cách ly, 0 rò rỉ.

**Chủ dự án (02/11/2026):** "Dùng thật sau". Tiếp tục xây phần kế tiếp và giao triển khai luôn.

**Lượt này:**

- **3.3 Thi lên cấp và chứng nhận.**
- **3.4 Hồ sơ năng lực** (tiến bộ của học viên theo thời gian).

Hai mảng này dùng lại trực tiếp bàn cờ và bộ chấm của `dsc-chess`, và hoàn thành lộ trình 6 cấp Tốt → Vua.

**Để sau, sẽ có kế hoạch riêng:**

- 2.6 kích hoạt phụ huynh hàng loạt (làm khi bắt đầu dùng thật);
- 3.5 hợp nhất app cũ;
- 3.6 `kids-player`;
- 3.7 khoá học online trả phí.

Dự án "Dương Sinh Studio" (`docs/plans/2026-11-01-nocobase-studio-*`) là một nhánh riêng, không thuộc kế hoạch này.

## 2. Giả định mặc định

Chủ dự án chưa chốt các điểm dưới đây. Implementer làm theo mặc định; chủ dự án đổi được sau mà không phải làm lại nhiều.

| #   | Điểm                  | Mặc định                                                                                                           | Lý do                                                                                          |
| --- | --------------------- | ------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------------- |
| G1  | Thi ở đâu             | **Tại lớp, trên máy của giáo viên/trung tâm.** Giáo viên mở "Phòng thi", chọn em, em làm bài dưới sự giám sát      | Trẻ không có tài khoản riêng. Thi tại nhà qua tài khoản phụ huynh thì không bảo đảm trung thực |
| G2  | Cấu trúc đề           | N thế cờ, **mỗi thế chỉ một lượt**: nước sai đầu tiên là trượt thế đó. Không gợi ý, có giới hạn thời gian toàn bài | Đúng chuẩn thi. Dùng lại bộ chấm `move`                                                        |
| G3  | Điểm đạt              | Mặc định **70%** số thế cờ đúng, cấu hình được theo từng đề                                                        | Phổ biến, dễ chỉnh                                                                             |
| G4  | Phần thực hành        | Tuỳ chọn theo đề: giáo viên chấm **Đạt / Không đạt** kèm nhận xét (ván đấu thực hành, kiến thức lý thuyết)         | Chuẩn đầu ra có phần không đo được bằng thế cờ                                                 |
| G5  | Ai quyết định lên cấp | **Quản lý duyệt.** Sau khi duyệt, hệ thống tự cập nhật cấp, cấp chứng nhận và ghi lịch sử                          | Tránh lên cấp nhầm. Có người chịu trách nhiệm                                                  |
| G6  | Chứng nhận            | Số chứng nhận tự sinh (`CN` + 5 số). In **A4 ngang từ trình duyệt** (ra PDF qua hộp thoại in)                      | Bản mã nguồn mở không có plugin in                                                             |
| G7  | Thưởng                | Đạt cấp được cộng **10 sao** (`loai = thuong_len_cap`)                                                             | Gắn với thẻ sao sẵn có                                                                         |
| G8  | Thông báo phụ huynh   | Trong app. Hàng đợi Zalo có thêm loại `dat_cap`, nhưng chỉ gửi khi đã có mẫu ZNS và chế độ Zalo được bật           | Zalo đang tắt                                                                                  |

## 3. Thiết kế 3.3 — Thi lên cấp

### 3.1 Dữ liệu (`setup/schema/phase3.mjs`; `collections` tự chạy tiếp `roles`)

| Collection        | Trường chính                                                                                                                                                                                                                                                      | Quan hệ                                                                                                                      |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `dsc_de_thi`      | `ma` (sequence `DT`), `ten`, `thoi_gian_phut` (mặc định 20), `diem_dat_phan_tram` (mặc định 70), `co_thuc_hanh` (checkbox), `huong_dan_thuc_hanh` (text), `trang_thai` (`nhap` / `xuat_ban`)                                                                      | m2o `cap_do` (**cấp thi lên**), m2m `the_co` (bảng nối `dsc_de_thi_the_co`)                                                  |
| `dsc_ky_thi`      | `ten`, `ngay`, `trang_thai` (`sap_dien_ra` / `dang_thi` / `da_dong`)                                                                                                                                                                                              | m2o `de_thi`, m2o `co_so`, m2o `giam_thi` (`dsc_nhan_su`), m2m `hoc_vien` (danh sách dự thi, bảng nối `dsc_ky_thi_hoc_vien`) |
| `dsc_bai_thi`     | `trang_thai` (`chua_thi` / `dang_thi` / `da_nop` / `cho_duyet` / `dat` / `khong_dat`), `bat_dau_luc`, `het_gio_luc`, `nop_luc`, `so_dung`, `so_cau`, `diem_phan_tram`, `thuc_hanh` (`chua_cham` / `dat` / `khong_dat` / `khong_ap_dung`), `nhan_xet`, `duyet_luc` | m2o `ky_thi`, m2o `hoc_vien`, m2o `nguoi_duyet` (`dsc_nhan_su`), m2o `chung_nhan`                                            |
| `dsc_chung_nhan`  | `so` (sequence `CN`), `ngay_cap`, `ho_ten_tren_chung_nhan` (chụp lại tên lúc cấp), `ghi_chu`                                                                                                                                                                      | m2o `hoc_vien`, m2o `cap_do`, m2o `bai_thi`                                                                                  |
| `dsc_lich_su_cap` | `ngay`, `ly_do` (`thi_dat` / `nhap_tay` / `nhap_lieu_cu`)                                                                                                                                                                                                         | m2o `hoc_vien`, m2o `cap_cu`, m2o `cap_moi`, m2o `bai_thi`                                                                   |

- `dsc_lam_bai` thêm m2o `bai_thi`. Lượt làm trong bài thi có `giao_bai = NULL`. Khoá duy nhất áp theo `(hoc_vien, the_co, bai_thi)` khi có `bai_thi`, kiểm ở plugin.
- `dsc_bai_thi` là duy nhất theo `(ky_thi, hoc_vien)`.

**Phân quyền:**

- **Quản lý:** toàn quyền với cả 5 bảng.
- **Giáo viên:** sửa được `dsc_de_thi`, `dsc_ky_thi`. Với `dsc_bai_thi` chỉ xem, và chỉ sửa `thuc_hanh` + `nhan_xet`. **Không** được duyệt, không tạo được `dsc_chung_nhan` hay `dsc_lich_su_cap`.
- **Kế toán, tư vấn:** chỉ xem chứng nhận.
- **Phụ huynh:** chỉ xem `dsc_bai_thi` ở trạng thái `dat` / `khong_dat`, `dsc_chung_nhan` và `dsc_lich_su_cap` **của con mình**. Cột tường minh, **không** thấy đề, thế cờ của đề hay lời giải.
  - Cập nhật `apply.mjs` và chạy `gen-parent-columns`, vì guard đóng mặc định.
- Thêm cả 5 bảng vào `parent-isolation.mjs` và `parent-oracle.mjs`. Thêm vai trò vào `staff-roles-check.mjs`.

### 3.2 Plugin `dsc-chess` 0.2.0: chế độ thi

Các action mới (kiểm quyền bên trong; người gọi phải là giáo viên hoặc quản lý; kỳ thi phải ở `dang_thi`; học viên phải có trong danh sách dự thi):

- `examStart({ kyThiId, hocVienId })`
  - Tạo hoặc lấy `dsc_bai_thi`.
  - Lần đầu thì đặt `bat_dau_luc` và `het_gio_luc = bat_dau + thoi_gian_phut`.
  - Trả danh sách thế cờ của đề theo thứ tự xáo cố định (seed = id bài thi) gồm `{ id, fen, de_bai, ben_di }`, kèm thời gian còn lại. **Không lời giải.**
  - Bài thi đã nộp: 409.
- `examMove({ baiThiId, theCoId, uci })`
  - Hết giờ: tự nộp, trả 409.
  - Thế đã kết thúc: 409.
  - Dùng lại lõi chấm của `move`, nhưng **sai một nước là trượt thế đó**: `ket_qua = sai`, trả `{ dung: false, ket_thuc: true }`. Đúng thì trả nước đáp; xong thế thì `dung`.
  - Không có `hint`.
- `examSubmit({ baiThiId })` (giáo viên bấm "Nộp bài", hoặc tự gọi khi hết giờ / khi làm hết thế cờ)
  - Tính `so_dung`, `so_cau`, `diem_phan_tram`; đặt `trang_thai = da_nop`.
  - Nếu đề không có phần thực hành thì `thuc_hanh = khong_ap_dung`.
  - Chạy được nhiều lần, kết quả như nhau.
- `examApprove({ baiThiId, ketQua: 'dat' | 'khong_dat' })` (**chỉ quản lý**)
  - Điều kiện: đã nộp và phần thực hành đã chấm (hoặc không áp dụng).
  - `dat` chỉ hợp lệ khi `diem_phan_tram ≥ diem_dat` và thực hành `dat` / `khong_ap_dung`.
  - Trong **một transaction**:
    - cập nhật `dsc_hoc_vien.cap_do` = cấp của đề;
    - tạo `dsc_lich_su_cap` và `dsc_chung_nhan`;
    - cộng 10 sao qua repository (F5 tự tính);
    - gắn `chung_nhan` vào bài thi; đặt `dat`, `duyet_luc`, `nguoi_duyet`.
  - Gọi lại không tạo thêm bản ghi.
- `certificate({ chungNhanId })`
  - Quản lý, giáo viên, kế toán: luôn được xem.
  - Phụ huynh: chỉ chứng nhận của con mình.
  - Trả dữ liệu để in: số, họ tên trên chứng nhận, cấp, ngày cấp, cơ sở, chuẩn đầu ra của cấp.
- **Hết giờ kiểm ở server.** Có thêm workflow hằng giờ (hoặc chạy trong FN) tự nộp các bài `dang_thi` đã quá `het_gio_luc`.
- **Unit test:**
  - sai một nước thì trượt thế;
  - hết giờ thì 409 và tự nộp;
  - không lộ lời giải;
  - giáo viên không duyệt được;
  - duyệt `dat` khi chưa đủ điểm thì 400;
  - duyệt lặp không tạo trùng;
  - xáo thứ tự cố định theo seed.

### 3.3 Workflow (`setup/workflows/apply.mjs`)

- **W15:** khi bài thi chuyển sang `da_nop` và `diem_phan_tram ≥ diem_dat`, đặt `cho_duyet` (thực hành đã chấm hoặc không áp dụng) rồi báo quản lý trong app. Khi giáo viên chấm thực hành sau, kiểm lại điều kiện này.
- **W16:** bài thi chuyển sang `dat` / `khong_dat` thì thông báo trong app cho phụ huynh (kênh `dsc_in_app`, người nhận là `tai_khoan` của phụ huynh em đó).
  - Nếu `dat` và Zalo có `ma_mau_dat_cap`: thêm dòng `dat_cap` vào `dsc_zalo_hang_doi`, khoá chống trùng `dat_cap:<bai_thi>:<ph>`.
  - Thêm option `dat_cap` cho hàng đợi Zalo và cấu hình `ma_mau_dat_cap`, theo mẫu `setup/workflows/zalo.mjs`.

### 3.4 Giao diện

- **Desktop**, nhóm mới "Thi cấp" (quản lý + giáo viên):
  - **Đề thi:** chọn thế cờ, có bàn cờ xem trước bằng khối `xem` sẵn có.
  - **Kỳ thi:** danh sách dự thi, nút "Mở thi" / "Đóng thi".
  - **Bài thi:** lọc `cho_duyet`. Nút "Duyệt: Đạt" / "Duyệt: Không đạt" (khối JS gọi `examApprove`, chỉ hiện cho quản lý). Cột điểm, thực hành.
  - **Chứng nhận:** nút "In chứng nhận".
- **Khối in chứng nhận** (`setup/ui/certificate.mjs`):
  - Gọi `certificate`, dựng HTML A4 ngang trong iframe ẩn, rồi `print()`.
  - Theo nhận diện Dương Sinh: màu chủ đạo `#2B3990`, font Roboto (nạp qua `importAsync` CSS hoặc dùng font hệ thống dự phòng), hoạ tiết ô cờ ở viền.
  - Nội dung: tên học viên, cấp đạt được, chuẩn đầu ra, số chứng nhận, ngày cấp, chữ ký "Giám đốc" (để trống ký tay).
  - Không nhúng ảnh từ dịch vụ ngoài trừ font.
- **Mobile giáo viên**, trang "Phòng thi":
  - chọn kỳ thi đang mở, rồi chọn em (hiện trạng thái từng em);
  - **khối thi** dựa trên khối `lam_bai`: đồng hồ đếm ngược theo `het_gio_luc` của server, thanh tiến độ "Câu 3/10", không có nút Gợi ý / Đi lại; sai thì chuyển câu; có nút "Nộp bài" (hỏi xác nhận);
  - sau khi nộp: hiện điểm, form chấm thực hành và nhận xét.
- **Mobile phụ huynh:** "Con tôi" thêm mục "Thi cấp & chứng nhận": các bài thi đã có kết quả, chứng nhận kèm nút "Xem / In", lịch sử cấp. **Không thêm tab mới** (phụ huynh đang có 5/6 tab).
- Thanh tab giáo viên đang có 4. Thêm "Phòng thi" là 5 (≤ 6).

## 4. Thiết kế 3.4 — Hồ sơ năng lực

- **Action `dscChess:profile({ hocVienId, thang = 6 })`.**
  - Phụ huynh: chỉ con mình. Nhân viên: mọi em.
  - Tính ở server **bằng repository/SQL có tham số trong plugin, KHÔNG dùng SQL chart phía client cho phụ huynh**, vì SQL chart bỏ qua ACL.
  - Dữ liệu trả về:
    - lịch sử cấp (timeline);
    - chuyên cần theo tháng: số buổi có mặt / số buổi của lớp em tham gia;
    - điểm trung bình 3 tiêu chí (`tu_duy`, `tap_trung`, `tinh_than`) theo tháng;
    - số thế cờ đúng theo tuần (8 tuần);
    - tổng sao và sao theo tháng;
    - kết quả thi.
  - **Không trả văn bản nhận xét nội bộ.**
- **Khối "Tiến bộ"** (`setup/ui/profile.mjs`): biểu đồ bằng SVG/CSS tự vẽ, không thêm thư viện. Có các phần:
  - dải thời gian 6 cấp, tô các cấp đã đạt kèm ngày;
  - cột chuyên cần theo tháng;
  - đường 3 tiêu chí;
  - cột bài tập theo tuần.

  a11y: mỗi biểu đồ có `role="img"`, `aria-label` tóm tắt số liệu, và bảng số liệu ẩn dành cho trình đọc màn hình.

- **Nơi hiển thị:**
  - desktop: popup học viên, thêm tab "Tiến bộ";
  - mobile giáo viên: tra cứu học viên;
  - mobile phụ huynh: "Con tôi", thêm phần "Tiến bộ".
- **Dữ liệu cấp hiện có:** nhiều em chưa có cấp (bàn giao: 122 em đang theo học thiếu cấp). Lịch sử cấp bắt đầu từ lúc có hệ thống thi; `nhap_tay` dùng khi quản lý đặt cấp bằng tay.
  - Thêm workflow **W17**: `dsc_hoc_vien.cap_do` đổi mà không qua `examApprove` thì ghi `dsc_lich_su_cap` với `ly_do = nhap_tay`.
  - Không tạo lịch sử giả cho dữ liệu cũ.

## 5. Thứ tự triển khai (mỗi bước: dev → verify → sao lưu → production → verify + web-check → commit)

1. **3.3-1** schema + ACL + `gen-parent-columns` + bộ dò (isolation, oracle, staff).
2. **3.3-2** `dsc-chess` 0.2.0: các action thi + unit test. Build, rồi `verify.mjs plugins` (bài học ba81131: phải có `client.js` / `server.js`).
3. **3.3-3** W15, W16, W17, tự nộp khi hết giờ, loại Zalo `dat_cap`.
4. **3.3-4** giao diện: desktop "Thi cấp", khối in chứng nhận, mobile "Phòng thi", phần "Thi cấp & chứng nhận" trong "Con tôi".
5. **3.4-1** action `profile` + khối "Tiến bộ" ở 3 nơi.
6. **Kiểm chứng** mục 6. Chụp ảnh Playwright: phòng thi, chứng nhận (PDF/ảnh), tiến bộ. Cập nhật mục "Kết quả" cuối file này, Tiến độ của kế hoạch chiến lược, README và file bàn giao.

**Quy tắc bắt buộc** (từ các sự cố 01/11):

- Kiểm bằng **tài khoản thử của từng vai trò**, không chỉ bằng root.
- Sau mỗi lần cài plugin: `verify.mjs plugins` + `web-check.mjs`.
- `collections` luôn kéo theo `roles`.
- Thêm trường cho phụ huynh xem: sửa `apply.mjs` + chạy `gen-parent-columns`.
- Không dùng tài khoản phụ huynh hay giáo viên thật. Dữ liệu `[TEST]` phải được dọn sau khi kiểm.
- Không gửi Zalo.

## 6. Kiểm chứng (`verify.mjs` mục 13, `setup/exam-check.mjs`)

1. Giáo viên mở kỳ thi rồi `examStart` cho em trong danh sách: được. Em ngoài danh sách: 403. Phụ huynh gọi bất kỳ action thi nào: 403.
2. Phản hồi của `examStart` / `examMove` không có lời giải. Phụ huynh không đọc được `dsc_de_thi` hay thế cờ của đề qua list, get, appends, route, lọc, sắp xếp.
3. Sai một nước thì thế đó `sai`, đi tiếp trên thế đó trả 409. Đúng hết thì `dung`.
4. Hết giờ (đặt `thoi_gian_phut` nhỏ, hoặc chỉnh `het_gio_luc` trong test) thì `examMove` trả 409 và bài tự nộp.
5. Nộp bài: điểm đúng. Đạt ngưỡng thì `cho_duyet` (W15). Có phần thực hành thì phải chấm rồi mới `cho_duyet`.
6. Giáo viên gọi `examApprove`: 403. Quản lý duyệt `dat` khi chưa đủ điểm: 400.
7. Quản lý duyệt `dat`:
   - cấp của học viên cập nhật;
   - có 1 chứng nhận (số `CN…`) và 1 dòng lịch sử cấp;
   - +10 sao, `tong_sao` tăng;
   - phụ huynh nhận thông báo trong app.
   - Duyệt lần hai không tạo thêm gì.
8. Phụ huynh A gọi `certificate` / `profile` cho con của phụ huynh B: 403. Gọi cho con mình: đúng dữ liệu, không có nhận xét nội bộ.
9. Quản lý sửa tay `cap_do`: W17 ghi lịch sử `nhap_tay`.
10. Số liệu của `profile` khớp phép đếm SQL độc lập trên cùng dữ liệu thử (chuyên cần, bài tập/tuần, sao).
11. Trình duyệt (Playwright, iPhone 13):
    - giáo viên thử thi cho em thử: đúng 1 thế, sai 1 thế, nộp, chấm thực hành;
    - quản lý duyệt trên desktop;
    - in chứng nhận: chụp ảnh khung in;
    - phụ huynh thử thấy chứng nhận và phần "Tiến bộ".

    0 lỗi JS.

## 7. Rủi ro

- **Gian lận thi:** chỉ thi trên máy giáo viên. Mỗi thế một lượt. Hết giờ kiểm ở server. Lời giải không bao giờ rời server.
- **Lên cấp nhầm:** quản lý duyệt, có điều kiện điểm, transaction, chạy lặp không đổi kết quả, có lịch sử cấp.
- **Lộ dữ liệu trẻ em qua hồ sơ / chứng nhận:** action kiểm quyền cha–con. Guard đóng mặc định. Có bộ dò cách ly và oracle cho bảng mới. Không có SQL chart cho phụ huynh.
- **Chứng nhận in xấu hoặc khác nhau giữa trình duyệt:** dùng HTML/CSS đơn giản, kiểm trên Chrome. Khổ A4 ngang qua `@page`.
- **Dev SQLite chập chờn:** chạy từng mục. Bản chạy đầy đủ chính thức là PostgreSQL production.

## 8. Việc của chủ dự án (khi bắt đầu dùng thật)

- Xác nhận hoặc đổi các giả định G1–G8.
- Soạn đề thi cho từng cấp từ ngân hàng thế cờ.
- Duyệt mẫu chứng nhận: chữ ký, con dấu, logo.
- Tạo mẫu ZNS "đạt cấp" nếu muốn gửi qua Zalo.
- Bổ sung cấp hiện tại cho các em còn thiếu (122 em).

## 9. Kết quả (02/11/2026, dev + production)

Các bước 3.3-1 → 3.3-4 và 3.4-1 đã xong. Mã ở repo `D:\code\duongsinh-nocobase`; plugin `@duongsinh/plugin-dsc-chess` bản 0.2.3.

**3.3-4 giao diện thi cấp** (commit `bddc207`):

- Desktop nhóm "Thi cấp" (quản lý + giáo viên): Đề thi (xem trước thế cờ), Kỳ thi (Mở thi / Đóng thi), Bài thi (chấm thực hành; khối "Duyệt: Đạt / Không đạt" chỉ quản lý, vai trò lấy từ `dscChess:me`), Chứng nhận (xem trước + In).
- `setup/ui/certificate.mjs`: HTML A4 ngang, màu #2B3990, Roboto (dự phòng font hệ thống), viền ô cờ bằng CSS, in qua iframe ẩn; PDF mẫu `reports/production/thi-cap-chung-nhan-a4.pdf`.
- Mobile giáo viên: tab "Phòng thi" (tab thứ 5), đồng hồ theo `het_gio_luc` của server, "Câu x/N", không Gợi ý/Đi lại, Nộp bài có xác nhận, chấm thực hành. Phụ huynh: mục "Thi cấp & chứng nhận" trong "Con tôi" (không thêm tab).
- Action mới của plugin: `examRoom`, `examGrade`, `examSummary`, `me`. W17: em chưa từng có cấp → `nhap_lieu_cu`, đổi cấp → `nhap_tay`.

**3.4-1 hồ sơ năng lực:**

- Action `dscChess:profile({ hocVienId, thang = 6 })`: tính ở server (Sequelize có tham số), phụ huynh chỉ con mình (403 chéo gia đình), nhân viên mọi em, không trả nhận xét. Trả về: dải 6 cấp (kèm ngày từ lịch sử cấp, không bịa ngày cho dữ liệu cũ), chuyên cần theo tháng (có mặt gồm học bù / số dòng điểm danh trong tháng, buổi huỷ bị loại), trung bình 3 tiêu chí theo tháng, thế cờ luyện tập đúng theo 8 tuần (giờ Việt Nam, tuần bắt đầu thứ Hai; lượt làm trong bài thi không tính), sao theo tháng + tổng, kết quả thi đã có kết quả cuối.
- Khối "Tiến bộ" (`setup/ui/profile.mjs`): SVG/CSS tự vẽ, mỗi biểu đồ có `role="img"` + `aria-label` + bảng số liệu ẩn, trạng thái rỗng thân thiện. Đặt ở popup học viên desktop (khối thêm vào popup, không phải tab riêng vì crudPage chưa hỗ trợ tab trong popup), mobile giáo viên (Tra cứu học viên), mobile phụ huynh (Con tôi, không thêm tab).
- Chứng nhận: cấp chưa có chuẩn đầu ra thì hiện câu mặc định; thêm dòng "Hà Nội, ngày … tháng … năm …" trên chỗ ký.

**Kiểm chứng:** 84 test đơn vị (`setup/tests/*.test.mjs`); `verify.mjs exam` mục 13.12 (cách ly `profile`, không lộ nhận xét, số liệu khớp phép đếm độc lập qua REST); Playwright desktop + iPhone 13 bằng tài khoản thử: `setup/ui-shots-exam.mjs` (luồng thi → duyệt → in → phụ huynh, 21 phép) và `setup/ui-shots-profile.mjs` (18 phép), 0 lỗi JS. Ảnh: `reports/` và `reports/production/` (`thi-cap-*.png`, `tien-bo-*.png`).

**Lưu ý vận hành:**

- Bản dev SQLite treo khi chạy bộ kiểm nặng (`verify.mjs exam`, 30 phút không ra kết quả); các phép kiểm tích hợp chính thức chạy trên production PostgreSQL sau khi sao lưu. Ảnh của hai script trên chụp từ production.
- Phép kiểm `staff-roles-check` chấp nhận bài thi `da_nop` hoặc `cho_duyet` vì W15 tự chuyển trạng thái.
