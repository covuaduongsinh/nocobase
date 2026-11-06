# Kế hoạch: Bộ máy tính toán tự động cho số liệu Dương Sinh (thay các cột công thức của Lark)

Đã duyệt ngày 31/10/2026. Mã nằm ở repo `D:\code\duongsinh-nocobase`.

## 1. Bối cảnh — trả lời câu hỏi "đã tính thật hay chỉ chép kết quả?"

**Đã tính thật (workflow chạy khi có thay đổi):**

- Gói học phí: `so_buoi_da_hoc`, `so_buoi_con_lai`, trạng thái Đang học/Sắp hết/Hết buổi (W1, khi tạo/sửa điểm danh); `da_thu` (W4, khi tạo/sửa phiếu thu); điền số buổi từ gói (W0, khi tạo gói); "Hết hạn" (W5, hằng ngày).
- Học viên: `tong_sao` (W2 tạo giao dịch sao khi điểm danh có sao, W3 cộng tổng).
- Phát sách: giá và thành tiền (W7, chỉ khi tạo mới).
- Dashboard: số liệu theo tháng/năm, chuyên cần… tính trực tiếp bằng SQL khi mở trang (thay các cột Tháng/Năm của Lark).

**Chỉ chép kết quả từ Lark (không tự cập nhật), hoặc chưa có:**

- `so_buoi_da_hoc_truoc` (số dư đầu kỳ, hợp lý là giá trị nhập); `hoc_phi` của gói cũ = số tiền đã ghép lúc nhập.
- `thanh_tien` của 338 lần phát sách chép từ Lark; sửa giá/ưu đãi sau khi tạo **không** tự tính lại.
- **Chưa có**: công nợ / trạng thái thanh toán của gói; tổng số buổi còn lại của học viên (Lark "số buổi tồn update"); số dư tiền sách (Lark "Deal book"); sĩ số lớp; ngày học gần nhất; nhắc sinh nhật (Lark "Nhắc sinh nhật"); đơn giá 1 buổi (Lark "HP 1 buổi").

**Lỗ hổng của các workflow hiện có (số liệu có thể sai sau này):**

1. **Xoá** điểm danh / phiếu thu / giao dịch sao không kích hoạt tính lại (chỉ có trigger tạo/sửa).
2. **Sửa gói** (số buổi, số buổi đã học trước, học phí, giảm giá) không tính lại số còn lại / công nợ.
3. Chuyển điểm danh hoặc phiếu thu từ gói A sang gói B: chỉ gói B được tính lại.
4. Sửa `sao_thuong` của điểm danh đã có không cập nhật giao dịch sao.
5. W4 cộng cả phiếu tiền sách vào "đã thu" của gói (cần lọc chỉ học phí).
6. Điểm danh "Có mặt" nhập tay (không qua nút ▶) mà không chọn gói → không trừ buổi.
7. Logic công thức nằm rải rác ở 6 workflow + `recalc.mjs` (JS) → dễ lệch nhau.

**Mục tiêu:** mọi cột tính toán có **một công thức duy nhất**, **tự cập nhật** khi tạo / sửa / xoá / nhập Excel, có **tự sửa sai hằng đêm** và **nút "Tính lại toàn bộ"**, và có kiểm thử chứng minh.

**Quyết định của chủ dự án (31/10/2026):**

- Số dư tiền sách: **chốt số dư đầu kỳ** — chỉ tính các lần nộp/phát sách **sau ngày chốt**, cộng số dư đầu kỳ do kế toán nhập.
- Công nợ: **gói nhập từ Lark coi là đã đủ** (không theo dõi công nợ); gói mới từ nay tự lấy học phí theo giá gói và tự tính công nợ.

## 2. Đối chiếu cột công thức / lookup của Lark → thiết kế mới

| Bảng Lark | Cột                                                      | Hiện tại                       | Thiết kế mới (tự tính)                                                          |
| --------- | -------------------------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------- |
| Deal HP   | Số buổi còn tồn = nộp − Σ đã học các năm − chưa thống kê | ✔ (W1) nhưng thiếu xoá/sửa gói | `so_buoi_con_lai` = `so_buoi` − `so_buoi_da_hoc`                                |
| Deal HP   | 2024/25/26 Tổng số buổi đã học (lookup)                  | ✔ đếm điểm danh                | `so_buoi_da_hoc` = đầu kỳ + đếm "Có mặt" gắn gói                                |
| Deal HP   | Đã hết / còn số buổi                                     | ✔ một phần                     | `trang_thai` tự tính (giữ Huỷ/Hết hạn nhập tay)                                 |
| Tổng Deal | HP 1 buổi                                                | chép vào ghi chú               | `don_gia_buoi` = (học phí − giảm giá) / số buổi                                 |
| Tổng Deal | Tổng HP + sách                                           | không lưu                      | báo cáo SQL theo lần nộp (không cần cột)                                        |
| Tổng Deal | Tháng/Năm/Ngày nộp                                       | SQL dashboard                  | giữ SQL                                                                         |
| HocVien   | số buổi tồn update (lookup)                              | **chưa có**                    | `hv.so_buoi_con_lai` = Σ còn lại các gói đang học                               |
| HocVien   | Nhắc sinh nhật                                           | **chưa có**                    | `hv.sinh_nhat_toi` (ngày sinh nhật kế tiếp) + bảng "Sinh nhật 7 ngày tới"       |
| Phát sách | Tiền sách = Giá − Ưu đãi                                 | ✔ chỉ khi tạo                  | `thanh_tien` tính lại cả khi sửa; `gia` mặc định theo sách                      |
| Deal book | Cộng dồn / đã dùng / tồn                                 | **chưa có**                    | `hv.so_du_tien_sach` = đầu kỳ + Σ tiền sách nộp sau chốt − Σ phát sách sau chốt |
| Nhận xét  | Tháng/Năm, số buổi (lookup)                              | SQL / quan hệ                  | giữ                                                                             |
| (mới)     | Công nợ gói                                              | **chưa có**                    | `cong_no`, `tt_thanh_toan`                                                      |
| (mới)     | Sĩ số lớp, ngày học gần nhất                             | **chưa có**                    | `lop.si_so`, `ngay_hoc_cuoi` (gói và học viên)                                  |

## 3. Thiết kế

### 3.1 Nguyên tắc

1. Tách **trường nhập** và **trường tự tính**. Trường tự tính có nhãn "(tự tính)", **không có trong form Thêm/Sửa**, chỉ hiện ở bảng/chi tiết.
2. **Một nguồn công thức**: toàn bộ công thức viết bằng SQL trong `setup/workflows/formulas.mjs`. Workflow dùng node **SQL** (`plugin-workflow-sql`, tham số an toàn `:hv`, không ghép chuỗi — đã kiểm `SQLInstruction.ts` dòng 46–61). SQL viết theo cú pháp chung PostgreSQL/SQLite (đã chứng minh với SQL dashboard); riêng phần ngày sinh nhật có 2 biến thể theo dialect.
3. **Đơn vị tính lại = một học viên** (`:hv`): tính lại mọi gói của học viên đó + các cột của học viên + sĩ số lớp của em. Nhờ vậy chuyển điểm danh/phiếu thu giữa 2 gói của cùng em luôn đúng cả hai gói, và chạy lại nhiều lần cho cùng kết quả.
4. Ba lớp bảo vệ: **sự kiện** (tạo/sửa/xoá) → **hằng đêm 02:00** tính lại toàn bộ (tự sửa sai, kể cả khi nhập Excel không kích hoạt hook) → **nút "Tính lại toàn bộ"** cho quản lý.
5. Thông báo không phụ thuộc "chuyển trạng thái" mà dùng **cờ** `thong_bao_sap_het_luc` (đã báo thì không báo lại; gia hạn thì xoá cờ) → không gửi trùng, không bỏ sót.

### 3.2 Trường mới (thêm vào `setup/schema/phase1.mjs`)

- `dsc_dang_ky_goi`: `cong_no` (money, tự tính), `tt_thanh_toan` (select: chua_thu / mot_phan / da_du / thua / khong_theo_doi, tự tính), `don_gia_buoi` (money, tự tính), `ngay_hoc_cuoi` (dateOnly, tự tính), `theo_doi_cong_no` (checkbox, mặc định true; gói có `lark_id` = false), `thong_bao_sap_het_luc` (datetime, kỹ thuật). Đổi nhãn `hoc_phi` → "Học phí phải nộp".
- `dsc_hoc_vien`: `so_buoi_con_lai` (tự tính), `ngay_hoc_cuoi` (tự tính), `sinh_nhat_toi` (dateOnly, tự tính hằng đêm), `so_du_tien_sach` (money, tự tính), `so_du_tien_sach_dau_ky` (money, nhập), `ngay_chot_tien_sach` (dateOnly, nhập; trống → không tính số dư).
- `dsc_lop_hoc`: `si_so` (integer, tự tính = số em "Đang học" trong lớp).
- Trạng thái "Chờ thanh toán" của gói chuyển sang `tt_thanh_toan`; `trang_thai` chỉ theo dõi buổi học (Đang học / Sắp hết / Hết buổi) + Huỷ / Hết hạn nhập tay.

### 3.3 Công thức (SQL, `formulas.mjs`) — tính cho học viên `:hv` (bản "toàn bộ" bỏ điều kiện `:hv`)

- Gói: `so_buoi_da_hoc = COALESCE(so_buoi_da_hoc_truoc,0) + COUNT(điểm danh trang_thai='co_mat' gắn gói)`; `so_buoi_con_lai = so_buoi − so_buoi_da_hoc` (NULL nếu chưa có số buổi); `da_thu = Σ phiếu thu trang_thai='da_thu' AND loai_thu='hoc_phi'` gắn gói; `cong_no = hoc_phi − COALESCE(giam_gia,0) − da_thu` khi `theo_doi_cong_no`; `tt_thanh_toan` theo cong_no/da_thu; `don_gia_buoi = (hoc_phi − giam_gia)/so_buoi`; `ngay_hoc_cuoi = MAX(ngày buổi "Có mặt")`; `trang_thai` = giữ nguyên nếu Huỷ/Hết hạn, ngược lại ≤0 → Hết buổi, ≤2 → Sắp hết, còn lại → Đang học.
- Học viên: `tong_sao = Σ giao dịch sao`; `so_buoi_con_lai = Σ còn lại (>0) các gói Đang học/Sắp hết`; `ngay_hoc_cuoi`; `so_du_tien_sach = đầu kỳ + Σ phiếu tiền sách (ngay_thu > ngày chốt) − Σ phát sách thanh_tien (ngay > ngày chốt)` (NULL nếu chưa có ngày chốt).
- Lớp: `si_so = COUNT(học viên Đang học thuộc lớp)` (tính lại cho các lớp của em).
- Phát sách (một bản ghi `:id`): `gia = COALESCE(gia, sach.gia_ban)`, `thanh_tien = gia − COALESCE(uu_dai,0)`.
- Hằng đêm: `sinh_nhat_toi` (biến thể PG/SQLite), gói quá hạn → Hết hạn (thay W5).

### 3.4 Workflow (sinh từ `setup/workflows/apply.mjs`, cơ chế phiên bản theo dấu vân tay đã có)

| Mới                  | Kích hoạt                                                                   | Việc                                                                                                                                                                |
| -------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **F0** (thay W0)     | Tạo gói (đồng bộ)                                                           | Điền số buổi + **học phí theo giá gói** nếu trống → tính lại học viên                                                                                               |
| **F1** (thay W1, W2) | Điểm danh tạo/sửa (đổi trạng thái, gói, học viên, sao)                      | Nếu "Có mặt" mà chưa có gói → gắn gói đang học cũ nhất; đồng bộ giao dịch sao theo `sao_thuong` (tạo / sửa / xoá); tính lại học viên; kiểm tra cờ thông báo sắp hết |
| **F2**               | Điểm danh xoá                                                               | Xoá giao dịch sao kèm theo; tính lại học viên (dữ liệu bản ghi đã xoá có `hocVienId`)                                                                               |
| **F3** (thay W4)     | Phiếu thu tạo/sửa + xoá (2 workflow vì mode 5/6/7 bị cấm)                   | Tính lại học viên                                                                                                                                                   |
| **F4**               | Gói sửa (đổi số buổi, đầu kỳ, học phí, giảm giá, gói học, trạng thái) + xoá | Tính lại học viên                                                                                                                                                   |
| **F5** (thay W3)     | Giao dịch sao tạo/sửa + xoá                                                 | Tính lại học viên                                                                                                                                                   |
| **F6** (thay W7)     | Phát sách tạo/sửa (đồng bộ) + xoá                                           | Tính giá/thành tiền bản ghi; tính lại học viên (số dư sách)                                                                                                         |
| **F7**               | Học viên sửa (trạng thái, số dư đầu kỳ, ngày chốt, lớp)                     | Tính lại học viên + sĩ số                                                                                                                                           |
| **FN** (thay W5)     | Hằng ngày 02:00 (giờ VN, container `TZ=Asia/Ho_Chi_Minh`)                   | SQL tính lại toàn bộ + hết hạn + sinh nhật + gửi thông báo "sắp hết" còn thiếu                                                                                      |
| **FM**               | Nút "Tính lại toàn bộ" (custom-action, chỉ quản lý) trên Bảng điều khiển    | Như FN, chạy ngay                                                                                                                                                   |

Giữ W6 (tạo sẵn điểm danh). Workflow cũ W0–W5, W7: script **tắt** theo `key` (giữ lịch sử chạy), không xoá. Trong lúc nhập dữ liệu hàng loạt vẫn dùng cơ chế tạm tắt rồi chạy FM.

### 3.5 Giao diện (`setup/ui/pages.mjs`, `mobile-pages.mjs`)

- Bỏ trường tự tính khỏi form Thêm/Sửa (gói: đã học, còn lại, đã thu, công nợ…; phát sách: thành tiền; học viên: tổng sao, còn lại, số dư sách); thêm vào bảng/chi tiết với nhãn "(tự tính)".
- Thêm trường mới vào `FIELD_GROUPS` (bắt buộc với validator — bẫy đã biết).
- Bảng điều khiển: bảng "Gói còn công nợ", bảng "Sinh nhật 7 ngày tới", bảng "Học viên sắp hết buổi" (theo `hv.so_buoi_con_lai ≤ 2`), nút "Tính lại toàn bộ".
- Trang học viên: cột "Số buổi còn lại", "Số dư tiền sách"; trang lớp: cột "Sĩ số".

### 3.6 Dữ liệu hiện có

- Gói có `lark_id` → `theo_doi_cong_no = false` (theo quyết định), gói mới mặc định true.
- `so_buoi_da_hoc_truoc` giữ nguyên (số dư đầu kỳ từ Lark).
- Số dư tiền sách: để trống tới khi kế toán nhập `ngay_chot_tien_sach` + `so_du_tien_sach_dau_ky` (xuất file mẫu `reports/4-so-du-tien-sach-dau-ky.csv` gồm các em đang học + số liệu tham khảo từ Lark Deal book để điền nhanh, rồi nhập lại bằng script).
- Sau khi triển khai: chạy FM; đối chiếu lại Lark (kỳ vọng vẫn 309/313 gói khớp số buổi còn lại).

### 3.7 Công cụ kiểm tra độc lập

- `setup/recalc.mjs` đổi thành **`--check`** (mặc định): tính lại bằng JS theo cùng định nghĩa ở mục 3.3 và **so với giá trị trong DB**, in danh sách lệch (không ghi). `--apply` gọi workflow FM qua API. Đây là "người kiểm toán" thứ hai cho công thức SQL.

## 4. Các bước triển khai (sau khi duyệt; thử trước, rồi production)

**Phối hợp với phiên giai đoạn 2 đang chạy song song** (đang sửa cùng repo: `tai_khoan` trên phụ huynh, vai trò `dsc_phu_huynh` với danh sách trường được xem trong `setup/acl/apply.mjs`, `setup/parent-isolation.mjs`, `verify.mjs`):

- Không triển khai cùng lúc với phiên đó. Bắt đầu khi `git status` của `duongsinh-nocobase` sạch (phiên kia đã commit); đọc lại các file đã đổi trước khi sửa.
- Thêm trường tự tính mới vào danh sách trường phụ huynh được xem: `hv.so_buoi_con_lai`, `hv.ngay_hoc_cuoi`, gói `tt_thanh_toan`, `cong_no`, `ngay_hoc_cuoi` (con của họ; không có trường kỹ thuật `theo_doi_cong_no`, `thong_bao_sap_het_luc`, số dư tiền sách nội bộ chỉ khi chủ dự án muốn).
- Sau triển khai, chạy cả `parent-isolation.mjs` để chắc trường mới không làm lộ dữ liệu giữa các gia đình.
- Ghi chú vào file bàn giao để phiên giai đoạn 2 dùng F1–F7 (không còn W1/W3/W4) khi nối Zalo ZNS vào thông báo "sắp hết".

0. **Kiểm tra trước** (lệnh chỉ đọc): plugin `workflow-sql`, `workflow-custom-action-trigger` đã bật trên dev và production; `app:getInfo` (hoặc tương đương) cho biết dialect để chọn biến thể SQL ngày sinh; RAM VPS còn đủ. Bản dev local đã bị dừng do thiếu RAM — **hỏi chủ dự án trước khi bật lại**; nếu không bật được, thử trên production với dữ liệu `[TEST]` + sao lưu trước.
1. `formulas.mjs`: viết SQL tính lại theo học viên / toàn bộ / phát sách / hằng đêm (2 biến thể ngày sinh).
2. Schema: thêm trường mục 3.2; chạy `run.mjs collections`.
3. Workflow F0–F7, FN, FM theo mục 3.4; tắt W0–W5, W7; `run.mjs workflows`.
4. Giao diện mục 3.5; `run.mjs ui`.
5. `recalc.mjs --check`; mở rộng `verify.mjs` (mục 5).
6. Dữ liệu mục 3.6: đánh dấu gói Lark, chạy FM, `recalc --check` = 0 lệch, đối chiếu Lark.
7. Xuất mẫu số dư tiền sách đầu kỳ; khi kế toán gửi lại thì nhập.
8. Sao lưu, commit (Conventional Commits), cập nhật README, file bàn giao, mục Tiến độ; lưu kế hoạch này vào `docs/plans/`.

## 5. Kiểm chứng (thêm vào `setup/verify.mjs`, chạy trên dev và production)

1. Tạo gói 10 buổi → học phí tự lấy theo giá gói, công nợ = học phí, `tt_thanh_toan` = chưa thu.
2. 3 điểm danh "Có mặt" → đã học 3, còn 7; **xoá 1** → đã học 2, còn 8.
3. **Sửa gói** số buổi 10 → 12 → còn 10; sửa "đã học trước" = 5 → còn 5.
4. Chuyển 1 điểm danh sang gói thứ hai của cùng em → cả hai gói đúng ngay.
5. Điểm danh "Có mặt" nhập tay không chọn gói → tự gắn gói đang học cũ nhất.
6. Phiếu thu học phí một phần → `mot_phan`, công nợ đúng; đủ → `da_du`; **xoá phiếu** → công nợ quay lại; phiếu **tiền sách** không làm đổi "đã thu" của gói.
7. Sửa `sao_thuong` 2 → 5 → tổng sao +3; đặt về 0 / xoá điểm danh → giao dịch sao bị xoá.
8. Phát sách → giá theo sách, sửa ưu đãi → thành tiền đổi; số dư tiền sách đúng sau ngày chốt.
9. Còn ≤ 2 buổi → đúng **một** thông báo (chạy lại không báo thêm); gia hạn → cờ được xoá.
10. Sĩ số lớp, `hv.so_buoi_con_lai`, `ngay_hoc_cuoi`, `sinh_nhat_toi` đúng.
11. Chạy FM hai lần → không thay đổi gì; `recalc.mjs --check` = 0 lệch trên toàn bộ dữ liệu thật.

## 6. Rủi ro

- SQL cập nhật trực tiếp **bỏ qua hook/nhật ký thay đổi** của NocoBase cho các trường tự tính → chấp nhận (đây là trường tính toán); thông báo xử lý bằng cờ, không dựa vào hook.
- Sai khác cú pháp PostgreSQL/SQLite → chỉ dùng cú pháp chung; ngày sinh nhật tách 2 biến thể; kiểm trên cả hai.
- Workflow chạy song song cho cùng học viên → công thức là "tính lại toàn bộ" nên kết quả cuối vẫn đúng; FN hằng đêm sửa mọi sai lệch còn sót.
- Validator blueprint của NocoBase 2.2.36 (bẫy đã biết) khi thêm trường → cập nhật `FIELD_GROUPS`, thử trên dev/try-one trước.
- Tải VPS: tính theo học viên rất nhẹ; tính toàn bộ chỉ chạy 02:00 hoặc khi bấm nút.

## 7. Tiến độ

- [x] 31/10: Kiểm tra trước — `workflow-sql`, `workflow-custom-action-trigger`, `workflow-javascript`, `field-formula` đã bật trên production; dialect lấy qua `app:getInfo` (`database.dialect`); bảng trung gian lớp–học viên: `dsc_lop_hoc_hoc_vien("dscLopHocId","dscHocVienId")`.
- [x] Bước 1: `setup/workflows/formulas.mjs` (nguồn công thức duy nhất) + `setup/tests/formulas.test.mjs` — đạt toàn bộ trên SQLite (bộ nhớ) và PostgreSQL production (schema tạm, ROLLBACK).
- [x] Bước 5 (một phần): `setup/recalc.mjs --check` — kiểm toán chỉ đọc trên production: 0 lệch ở các trường hiện có (349 gói, 175 học viên, 338 phát sách, 7 lớp).
- [x] Bước 2: schema thêm các trường mục 3.2 trên production; quan hệ `dsc_giao_dich_sao.diem_danh` chuyển `onDelete: CASCADE`, vì NocoBase đặt `diemDanhId` = NULL **trước** khi workflow xoá chạy. Nếu không đổi, F2 không tìm được giao dịch sao để xoá.
- [x] Bước 3: workflow F0–F7 (gồm bản "khi xoá"), FN (02:00), FM (custom-action toàn cục) đã bật; W0–W5, W7 đã tắt (giữ lịch sử). Mọi lần chạy đều status 1.
- [x] Bước 4: giao diện. Trường tự tính bỏ khỏi form. Dashboard có thêm "Gói còn công nợ", "Học viên sắp hết buổi", "Sinh nhật sắp tới" và nút "Tính lại toàn bộ số liệu". Nút này gắn workflow bằng `flowModels:save`, vì blueprint/updateSettings không nhận `triggerWorkflows` cho nút toàn cục. Bẫy mới: khối details vượt khoảng 14–15 trường thì validator 2.2.36 báo nhầm `missing-default-field-groups`.
- [x] Bước 5: `verify.mjs` mục 8 (11 kịch bản) → production **109 đạt / 0 trượt**, gồm cách ly phụ huynh. `recalc.mjs --check` = **0 lệch**. `--apply` nay gọi FM.
- [x] Bước 6: 349 gói Lark đặt `theo_doi_cong_no = false`. Đã tính lại toàn bộ lần đầu bằng SQL. 23 gói đang "Sắp hết" được đánh dấu đã báo để không gửi thông báo dồn.
- [x] Bước 7: `setup/book-opening.mjs --export` → `reports/4-so-du-tien-sach-dau-ky.csv` (87 học viên đang học, kèm số tham khảo = đã nộp − đã phát). **Chờ kế toán** điền `ngay_chot`, `so_du_dau_ky`, rồi chạy `--import`.
- [x] Bước 8: sao lưu trước khi đổi; commit `9ca15fd`, `0a65f2d` (repo `duongsinh-nocobase`); README cập nhật.
- Ghi chú: học viên đổi `hoc_vien` trên điểm danh/phiếu thu thì chỉ học viên mới được tính lại ngay; học viên cũ được FN sửa lúc 02:00.
