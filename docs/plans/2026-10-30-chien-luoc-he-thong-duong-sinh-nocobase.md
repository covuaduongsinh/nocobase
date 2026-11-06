# Chiến lược & kế hoạch hệ thống phần mềm Dương Sinh trên NocoBase

Đã duyệt ngày 30/10/2026. Nguồn: hồ sơ năng lực Dương Sinh (bản cập nhật 9/2026).

## 1. Bối cảnh

**Dương Sinh hôm nay (theo hồ sơ năng lực 9/2026):** 3 địa điểm Hà Nội (trụ sở + hiệu sách Tôn Đức Thắng, CLB Kim Liên, CLB Vĩnh Phúc), 4 nhân sự đào tạo, lộ trình 6 cấp Tốt → Mã → Tượng → Xe → Hậu → Vua, gói học phí 10/20/40 buổi kèm quà, nhận xét sau mỗi buổi, thẻ sao đổi quà, liên lạc phụ huynh qua Zalo, 10+ đầu sách bán qua web/Shopee/Tiki, giải nội bộ hằng quý và DSC Open, dữ liệu rải rác trên Lark/Drive và vài app riêng (dsc.edu.vn, thuviencovua.com, app học cờ).

**Đích edu tech:** từ "CLB + hiệu sách" thành **tổ chức giáo dục dựa trên dữ liệu**: mỗi học viên có hồ sơ học tập số theo 6 cấp, phụ huynh thấy tiến bộ của con, học online/offline chung một lộ trình, mô hình chuẩn hoá đủ để **nhượng quyền** và **vào trường học**, AI hỗ trợ giáo viên và vận hành.

**Quyết định đã chốt với chủ dự án:**

- Giai đoạn 1 ưu tiên **vận hành CLB**.
- NocoBase là **nền tảng duy nhất** (thay thế toàn bộ): backoffice + cổng phụ huynh/học viên + học liệu; các app cũ được hợp nhất dần.
- Hạ tầng: **VPS + Docker + PostgreSQL**.
- **Ưu tiên mã nguồn mở**: chỉ plugin miễn phí trong repo; thiếu thì tự viết plugin riêng.

## 2. Chiến lược

### 2.1 Nguyên tắc

1. **Dữ liệu học viên là tài sản lõi.** Một hồ sơ học viên duy nhất xuyên suốt: tuyển sinh → học → thi cấp → giải đấu → mua sách.
2. **Cấu hình trước, code sau.** Dùng no-code của NocoBase (collection, block, workflow, ACL) cho ~80% nhu cầu; chỉ viết plugin khi thật sự thiếu (bàn cờ tương tác, thanh toán VietQR, Zalo OA).
3. **Không fork lõi NocoBase.** Mã riêng nằm trong plugin `@duongsinh/*` ở repo git riêng (clone vào `packages/plugins/@duongsinh/`), để nâng cấp NocoBase không xung đột.
4. **Thiết kế sẵn cho đa cơ sở / nhượng quyền** ngay từ đầu (trường `co_so` + phân quyền theo cơ sở), dù hiện mới có 2 CLB.
5. **Bảo vệ dữ liệu trẻ em**: tuân thủ Nghị định 13/2023/NĐ-CP và Luật Bảo vệ dữ liệu cá nhân (hiệu lực 01/01/2026): đồng ý của phụ huynh, phân quyền chặt, nhật ký truy cập, sao lưu mã hoá.

### 2.2 Kiến trúc mục tiêu

```
                    ┌──────────────── NocoBase (1 instance, Postgres) ───────────────┐
 Nhân sự  ── v1/v2 admin UI ──►  Phân hệ: CLB · CRM · Học phí · LMS 6 cấp · Giải đấu ·
 Phụ huynh ── mobile client ──►           Sách & học cụ · Nhân sự · Báo cáo · AI
 Học viên  ── mobile/web     ──►  Plugin @duongsinh/*: chess-board, vietqr, zalo-oa,
 Khách     ── public forms   ──►           level-exam, star-points
                    └────────────── REST API (api-keys) ───────────────────────────┘
                         ▲                         ▲
            Website covuaduongsinh.com      Shopee/Tiki (nhập đơn định kỳ)
```

- **Dữ liệu chính:** data source `main` (Postgres). App cũ có DB riêng → kết nối tạm qua data source ngoài (`plugin-data-source-manager`, `plugin-collection-fdw`) trong thời gian chuyển đổi, rồi hợp nhất.
- **Client:** cấu hình trang trên client v2 (`/v`, FlowEngine) cho trang mới nơi đã ổn định; phụ huynh/học viên dùng `plugin-mobile`.

### 2.3 Plugin có sẵn sẽ dùng (đều trong `packages/plugins/@nocobase/`)

| Nhu cầu                                         | Plugin                                                                                                                                   |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Phân quyền theo vai trò/cơ sở                   | `plugin-acl`, `plugin-users`, `plugin-departments`                                                                                       |
| Đăng nhập phụ huynh bằng SĐT                    | `plugin-auth-sms`, `plugin-verification`                                                                                                 |
| Lịch học, kanban lead, gantt dự án              | `plugin-calendar`, `plugin-kanban`, `plugin-gantt`                                                                                       |
| Tự động hoá (nhắc gia hạn, trừ buổi, thông báo) | `plugin-workflow` + `-action-trigger`, `-delay`, `-loop`, `-request`, `-manual`, `-notification`, `-mailer`, `-aggregate`, `-javascript` |
| Thông báo                                       | `plugin-notification-in-app-message`, `plugin-notification-email`                                                                        |
| Form đăng ký học thử / giải đấu công khai       | `plugin-public-forms`                                                                                                                    |
| Mã học viên, số phiếu thu                       | `plugin-field-sequence`, `plugin-field-formula`                                                                                          |
| Phiếu thu, chứng nhận cấp độ                    | `plugin-action-print`                                                                                                                    |
| Nhập dữ liệu cũ từ Excel/Lark                   | `plugin-action-import`, `plugin-action-export`                                                                                           |
| Dashboard                                       | `plugin-data-visualization`, `plugin-data-visualization-echarts`                                                                         |
| Học liệu, ảnh, video                            | `plugin-file-manager`, `plugin-file-previewer-office`, `plugin-block-markdown`                                                           |
| Nhận xét, trao đổi                              | `plugin-comments`, `plugin-block-comment`                                                                                                |
| App phụ huynh/học viên                          | `plugin-mobile`, `plugin-mobile-client`                                                                                                  |
| AI trợ lý giáo viên, MCP cho Claude             | `plugin-ai`, `plugin-mcp-server`                                                                                                         |
| Nhượng quyền (mỗi đối tác 1 app)                | `plugin-multi-app-manager`, `plugin-multi-app-share-collection`                                                                          |
| Sao lưu, nhật ký                                | `plugin-backups`/`plugin-backup-restore`, `plugin-audit-logs`, `plugin-logger`                                                           |
| Tiếng Việt                                      | `plugin-localization` (bổ sung bản dịch vi-VN còn thiếu)                                                                                 |

### 2.4 Plugin tự viết `@duongsinh/*` (theo thứ tự cần)

1. `plugin-dsc-star-points` — sổ cái thẻ sao (cộng/trừ/đổi quà), chống sửa tay. _(có thể làm bằng collection + workflow trước, chỉ thành plugin nếu cần chặt chẽ)_
2. `plugin-dsc-vietqr` — sinh mã VietQR theo phiếu thu, nhận webhook ngân hàng/cổng (Casso/SePay/PayOS) → workflow đánh dấu đã thu.
3. `plugin-dsc-zalo-oa` — kênh thông báo Zalo OA/ZNS gắn vào `notification-manager`.
4. `plugin-dsc-chess-board` — block bàn cờ tương tác (chessground + chess.js, FEN/PGN) cho bài tập, đề thi cấp, xem ván đấu; viết trên client v2 (FlowModel).
5. `plugin-dsc-level-exam` — đề thi lên cấp: ngân hàng thế cờ, chấm tự động, cấp chứng nhận.
6. `plugin-dsc-kids-player` — trình học cho trẻ 3–6 tuổi (audio + hoạt hình, không cần đọc). **Phần khó nhất của "thay thế toàn bộ"**; nếu trải nghiệm trong NocoBase mobile chưa đủ mượt, giữ một PWA nhẹ riêng nhưng **dữ liệu và tài khoản vẫn ở NocoBase** qua API.
7. `plugin-dsc-swiss-pairing` — xếp cặp hệ Thuỵ Sĩ cho giải đấu (hoặc nhập/xuất từ Swiss-Manager ở bản đầu).

## 3. Mô hình dữ liệu lõi (giai đoạn 1–2)

- **Tổ chức:** `co_so` (cơ sở/đối tác), `phong_hoc`, `nhan_su` (liên kết `users`, vai trò: CEO, giáo viên, trợ giảng, thực tập sinh, kế toán, bán hàng).
- **Khách hàng:** `phu_huynh` (SĐT, Zalo, đồng ý dữ liệu), `hoc_vien` (mã HV, ngày sinh, trường, cấp hiện tại, cơ sở) — n-n với phụ huynh.
- **Đào tạo:** `cap_do` (6 cấp, chuẩn đầu ra), `giao_trinh`/`bai_hoc` (theo cấp, gắn sách), `lop_hoc` (cấp, cơ sở, GV, lịch), `buoi_hoc`, `diem_danh`, `nhan_xet_buoi` (mẫu tiêu chí: tư duy, chiến thuật, tập trung, cảm xúc, hợp tác + văn bản), `ket_qua_thi_cap`.
- **Tài chính:** `goi_hoc` (10/20/40 buổi, giá, quà), `dang_ky_goi` (số buổi còn lại, hạn), `phieu_thu`, `qua_tang`, `the_sao` (sổ cái).
- **Tuyển sinh:** `lead` (nguồn, trạng thái, học thử), `hoc_thu`.
- **Giai đoạn sau:** `giai_dau`, `ky_thu_giai`, `van_dau`; `san_pham`, `ton_kho`, `don_hang`; `truong_doi_tac`, `hop_dong`; `bai_tap_online`, `tien_do_hoc`.

## 4. Lộ trình triển khai

### Giai đoạn 0 — Nền tảng (tháng 11/2026, ~3 tuần)

- Dựng VPS (4 vCPU/8 GB/100 GB SSD), Docker image NocoBase chính thức + PostgreSQL 16, HTTPS (Caddy/Nginx), domain `app.dsc.edu.vn`.
- Sao lưu tự động hằng ngày ra ngoài VPS (pg_dump + `storage/` lên Drive/S3), thử khôi phục 1 lần.
- Môi trường dev cục bộ theo `docs/plans/2026-10-30-chay-nocobase-dev.md`; môi trường staging trên VPS (2 container) để thử nâng cấp trước.
- Tạo repo `duongsinh-nocobase-plugins`; quy trình: cấu hình làm trên staging → migrate sang production bằng backup/restore hoặc `plugin-ui-templates` (cần thử nghiệm cơ chế chuyển cấu hình).
- Rà bản dịch vi-VN, bật Localization.

### Giai đoạn 1 — Vận hành CLB (12/2026 – 02/2027) ← ưu tiên

- Collection lõi ở mục 3 (tổ chức, khách hàng, đào tạo, tài chính).
- Nhập dữ liệu học viên/phụ huynh hiện có từ Lark/Excel.
- Màn hình giáo viên (mobile): danh sách lớp hôm nay → điểm danh → nhận xét buổi → cộng sao (≤ 2 phút/học viên).
- Workflow: điểm danh "có mặt" → trừ 1 buổi trong gói; còn ≤ 2 buổi → nhắc gia hạn; nhận xét lưu xong → thông báo phụ huynh; hết hạn gói → cảnh báo.
- Phiếu thu in được, theo dõi công nợ; `plugin-dsc-vietqr` bản đầu.
- Phân quyền: CEO thấy tất cả; giáo viên chỉ thấy lớp mình; kế toán thấy tài chính.
- Dashboard CEO: học viên đang học theo cơ sở/cấp, tỷ lệ chuyên cần, doanh thu tháng, gói sắp hết.
- **Thước đo:** 100% buổi học được điểm danh + nhận xét trong hệ thống; bỏ bảng tính song song sau 4 tuần chạy thật.

### Giai đoạn 2 — Tuyển sinh & cổng phụ huynh (03 – 05/2027)

- CRM lead: form công khai học thử, kanban trạng thái, nhắc chăm sóc, tỷ lệ chuyển đổi theo nguồn.
- Cổng phụ huynh (mobile, đăng nhập SĐT): lịch học, nhận xét từng buổi, số buổi còn, thẻ sao, thanh toán VietQR.
- `plugin-dsc-zalo-oa` thay tin nhắn Zalo cá nhân thủ công.
- **Thước đo:** ≥ 70% phụ huynh kích hoạt cổng; thời gian xử lý lead < 24h.

### Giai đoạn 3 — Edu tech: LMS 6 cấp (06 – 10/2027)

- Số hoá giáo trình theo cấp (bài học, video, liên kết sách), bài tập về nhà trên `plugin-dsc-chess-board`.
- Thi lên cấp (`plugin-dsc-level-exam`), chứng nhận in/PDF, hồ sơ năng lực học viên theo thời gian.
- Hợp nhất app học cờ / thuviencovua.com vào NocoBase; nhánh mầm non `plugin-dsc-kids-player`.
- Khoá học online trả phí (freemium: cấp Tốt miễn phí).
- **Thước đo:** % học viên làm bài tập online hằng tuần; số người dùng online trả phí.

### Giai đoạn 4 — Mở rộng (từ 11/2027)

- Giải đấu: đăng ký online, lệ phí, xếp cặp, kết quả, Elo nội bộ.
- Hiệu sách & học cụ: sản phẩm, tồn kho, đơn hàng; nhập đơn Shopee/Tiki định kỳ; xuất bản (kế hoạch 6 đầu sách/năm).
- Trường học đối tác: hợp đồng, lớp ngoại khoá, báo cáo cho nhà trường.
- Nhượng quyền: mỗi đối tác một sub-app (`plugin-multi-app-manager`), giáo trình/chuẩn dùng chung, báo cáo tổng.
- AI: trợ lý soạn nhận xét từ ghi chú ngắn, gợi ý bài tập theo điểm yếu, tóm tắt tiến bộ tháng cho phụ huynh (`plugin-ai`); `plugin-mcp-server` để Claude truy vấn dữ liệu vận hành.

## 5. Tổ chức thực hiện & rủi ro

- **Nhân lực:** CEO làm product owner; 1 người vận hành/cấu hình NocoBase (có thể là chính đội hiện tại sau đào tạo); lập trình plugin bằng Claude Code + 1 dev part-time. Chạy theo chu kỳ 12 tuần như văn hoá công ty, demo cuối mỗi 2 tuần.
- **Chi phí ước tính:** VPS ~0,5–1 triệu đ/tháng; SMS OTP/ZNS theo lượng; không phí bản quyền.
- **Rủi ro:**
  - NocoBase nâng cấp phá cấu hình/plugin → luôn thử trên staging, ghim phiên bản, đọc changelog.
  - Client v1/v2 đang chuyển tiếp → plugin mới viết cho v2; tránh phụ thuộc API nội bộ chưa ổn định.
  - Trải nghiệm trẻ 3–6 tuổi khó đạt trong UI admin → phương án PWA riêng dùng chung dữ liệu (đã nêu ở 2.4).
  - Đội ngũ nhỏ, ngại nhập liệu → thiết kế màn hình giáo viên tối giản, đo thời gian thao tác.
  - Dữ liệu trẻ em → phân quyền, nhật ký, đồng ý phụ huynh, sao lưu mã hoá.

## 6. Việc cần làm ngay sau khi duyệt

1. Ghi kế hoạch này thành `docs/plans/2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md`.
2. Các bước kế tiếp (mỗi bước một plan riêng trong `docs/plans/`): (a) chạy NocoBase dev cục bộ (plan đã có); (b) đặc tả chi tiết collection + workflow giai đoạn 1; (c) dựng VPS production.

## Tiến độ (cập nhật 30/10/2026)

- [x] Đặc tả giai đoạn 1: `docs/plans/2026-10-30-giai-doan-1-van-hanh-clb-dac-ta.md`.
- [x] Repo riêng `D:\code\duongsinh-nocobase` (git cục bộ, chưa có remote): script dựng 13 collection, dữ liệu danh mục, 4 vai trò, 6 workflow; chạy lặp an toàn; `setup/verify.mjs` đạt 8/8 kiểm tra trên bản dev SQLite.
- [x] Bộ triển khai VPS soạn sẵn (`deploy/`): NocoBase 2.2.36 + PostgreSQL 16 + Caddy + sao lưu hằng ngày. Chưa thuê VPS, chưa deploy.
- [x] Giao diện quản trị bằng script (`setup/ui/`): 6 nhóm menu, 13 trang, dashboard 4 biểu đồ SQL + bảng gói cần gia hạn, kanban lead, lịch buổi học; ngôn ngữ mặc định vi-VN. Dữ liệu mẫu `[DEMO]` và lệnh đối soát `recalc`.
- [x] Màn hình giáo viên trên mobile (`/v/mobile`): buổi hôm nay / 7 ngày qua, nút ▶ "Bắt đầu điểm danh" (W6 tạo sẵn "Có mặt" cho cả lớp), chấm & nhận xét từng em; tra cứu học viên. Quyền thấy menu theo vai trò.
- [x] Quy tắc trừ buổi chốt: chỉ "Có mặt" mới trừ.
- [ ] Bản dev giữ SQLite (chủ dự án không có mật khẩu `postgres`); lỗi khoá ghi khi nhiều workflow chạy song song xử lý bằng `recalc`. Production vẫn PostgreSQL.
- [x] Tạo tài khoản 4 nhân sự trên production (Trúc: Giáo viên + Quản lý; An, Phương Anh, Hà: Giáo viên), gắn hồ sơ `dsc_nhan_su`.
- [x] Nhập từ Lark Base (`HocVien`, `Contact`, `Level`): 179 học viên, 48 phụ huynh; giữ 6 cấp (NM→Tốt, CB/TC <quân>→<quân>); không nhập CCCD. Script `setup/import-lark.mjs` chạy lại an toàn theo `lark_id`.
- [ ] Bổ sung dữ liệu còn thiếu cho 127 em đang theo học: 52 thiếu ngày sinh, 96 chưa gắn phụ huynh, gần như chưa có cấp độ; 4 ngày sinh bất thường.
- [x] Nhập từ Lark: 7 lớp (`LopHoc`, không có danh sách học viên trong lớp) và 349 kỳ học phí (`Deal HP`) thành đăng ký gói; thêm `so_buoi_da_hoc_truoc` để số buổi còn lại khớp Lark (313/313 khớp). Không tạo phiếu thu vì Lark không lưu số tiền.
- [ ] Xếp học viên vào 7 lớp; gán giáo viên cho 5 lớp (Lark ghi "Phùng Đức Việt", "Andy Phạm" — chưa khớp nhân sự hiện tại).
- [ ] Rà 47/91 học viên "Đang học" không có gói còn buổi (thiếu kỳ học phí mới trên Lark hoặc trạng thái cũ).
- [x] Nhập từ Lark: lịch sử nhận xét 2024–2026 (625 buổi, 5.635 lượt có mặt, không trừ gói) và 7 lead; suy danh sách 7 lớp từ nhận xét 8 tuần gần nhất (một em có thể nhiều lớp); gộp 4 cặp hồ sơ trùng.
- [x] Quyết định: mọi nhân viên, giáo viên thấy tất cả các lớp (không giới hạn theo lớp).
- [ ] Chủ dự án xử lý 2 file rà soát trong `duongsinh-nocobase/reports/` (gói học phí còn thiếu, dữ liệu học viên còn thiếu).
- [ ] Báo lỗi validator NocoBase 2.2.36: `missing-default-field-groups` báo nhầm khi popup có 2 bảng liên quan (đang né bằng cách chỉ để 1 bảng).
- [x] Deploy production 30/10/2026: https://nocobase.dsc.edu.vn trên VPS 217.15.160.118 qua Dokploy (project `nocobase-dsc`), NocoBase 2.2.36 + PostgreSQL 16, HTTPS. Đã dựng đủ cấu trúc, workflow, giao diện; kiểm thử đầu-cuối 11/11 đạt trên PostgreSQL; SQL dashboard chạy đúng trên PostgreSQL. Không nạp dữ liệu mẫu.
- [x] Sao lưu tự động hằng ngày 02:30 giờ VN (pg_dump + storage, giữ 14 ngày, `/srv/backups/nocobase-dsc/` trên VPS); đã khôi phục thử thành công vào DB tạm.
- [x] Đẩy bản sao lưu ra Google Drive mỗi đêm (`NocoBase-DSC-backups`, giữ 30 ngày, quyền `drive.file`); đã tải về kiểm tra đọc được.
- [ ] Tạo OAuth client_id riêng cho rclone trên Google Cloud (client_id dùng chung của rclone sẽ bị Google ngừng trong 2026).
- [ ] Theo dõi tài nguyên VPS (lúc deploy chỉ còn ~1,5 GB RAM trống, NocoBase dùng ~700 MB).

- [ ] **Giai đoạn 2** (lập kế hoạch 31/10/2026): `docs/plans/2026-10-31-giai-doan-2-cong-phu-huynh-zalo-vietqr.md`; prompt cho phiên triển khai: `docs/plans/2026-10-31-prompt-trien-khai-giai-doan-2.md`. Đã chốt 5 quyết định (mục 6 kế hoạch): mật khẩu tạm, SePay, OA đã xác thực, phụ huynh chỉ xem nhận xét gửi PH + phiếu thu, thứ tự 2.1→2.5.
  - [x] **2.1 Cổng phụ huynh (31/10/2026)** — dev + production: `dsc_phu_huynh.tai_khoan`, vai trò `dsc_phu_huynh` (data scope theo tài khoản, cột tường minh, chỉ xem), 3 trang mobile (Con tôi / Nhận xét / Gói học phí), `setup/parents.mjs` (mặc định chỉ liệt kê). Phát hiện NocoBase 2.2.36 **không áp data scope cho appends lồng nhau và route quan hệ** → plugin `@duongsinh/plugin-dsc-guard` (cài trên production). Kiểm thử cách ly 2 gia đình trong `verify.mjs`: production "tất cả đạt" 78/78, không rò rỉ qua 322 phép dò. Commit `1fb2bd4` (duongsinh-nocobase).
  - [x] Cấp tài khoản phụ huynh thật (chủ dự án duyệt 31/10/2026): 22 tài khoản (mọi SĐT hợp lệ có con đang theo học); kiểm thử cách ly chạy lại trên production ngay trước đó, sao lưu trước. Danh sách gửi phụ huynh: `duongsinh-nocobase/reports/4-tai-khoan-phu-huynh.csv` (có mật khẩu tạm, không commit) — chủ dự án tự gửi qua Zalo.
  - [ ] Còn 26/48 hồ sơ phụ huynh chưa có tài khoản (15 thiếu/sai SĐT, còn lại không có con đang học); 92 em đang học chưa gắn phụ huynh → bổ sung rồi chạy lại `parents.mjs --all` (chỉ tạo tài khoản mới, không đổi mật khẩu cũ).
  - [x] **2.2 VietQR (31/10/2026)**: khối QR trong popup gói (desktop + mobile phụ huynh), VPBank 168686838 — DUONG SINH CHESS JSC, số tiền = `cong_no`, nội dung `DSC <mã đăng ký>`; ẩn khi không theo dõi công nợ / đã thu đủ. Chưa xem tận mắt trên giao diện (máy thiếu RAM để chạy client dev) — chủ dự án kiểm tra bằng một gói mới có công nợ.
  - [x] **2.3 Tuyển sinh (31/10/2026)**: form công khai → `dsc_dang_ky_web` → lead (W11), nhắc lead 24 giờ, nút "Chuyển thành học viên" (W12), biểu đồ nguồn. Commit `25ded42`.
  - [x] **2.4 Zalo ZNS (31/10/2026, mã + hạ tầng xong, chế độ `tat`)**: hàng đợi `dsc_zalo_hang_doi` + plugin `@duongsinh/plugin-dsc-zalo` (worker mỗi phút, token OA tự làm mới) + W9/W10 + mở rộng F1/FN/FM; `setup/zalo.mjs`; trang "Thông báo Zalo". Dev 147/0, production 157/0 (đã sao lưu trước). Commit `f69b6d5`. **Chờ chủ dự án**: App ID/secret, 3 mẫu ZNS, refresh token, SĐT thử (xem `duongsinh-nocobase/docs/zns-mau-de-xuat.md`), rồi bật `thu` → duyệt → `that`.
  - [x] **2.5 Webhook SePay (01/11/2026, mã + hạ tầng xong, chưa có khoá thật → endpoint 503)**: plugin `@duongsinh/plugin-dsc-bank-webhook`, sổ `dsc_giao_dich_ngan_hang`, W13 (báo chờ xử lý), W14 (ghép tay), trang "Giao dịch ngân hàng". Dev 189/0, production 173/0. Commit `b701122`. **Chờ chủ dự án**: đăng ký SePay + khoá API (xem `duongsinh-nocobase/docs/huong-dan-sepay.md`); trước khi bật thật xem mục "PHẢI xác minh" trong README.

- [x] **Giai đoạn 3 — LMS 6 cấp (01/11/2026, dev + production)**: `docs/plans/2026-11-01-giai-doan-3-lms-6-cap.md` (mục 12 "Kết quả"). Bài học / thế cờ / bài giao / lần làm bài, plugin `dsc-chess` chấm ở server, bàn cờ trên điện thoại cho phụ huynh, giáo viên giao bài + xem ai đã làm, số liệu tự tính (`si_so`, `so_da_xong`, `so_bai_tap_tuan`), dashboard, 20 thế cờ mẫu ở trạng thái nháp. Kèm hai sửa lỗi quan trọng: quyền đọc cột của vai trò nhân viên (`fields = []`) và cửa sổ sai quyền khi chạy `collections` trước `roles` (guard 0.1.5 đóng mặc định).
- [x] **Giai đoạn 3.3 / 3.4 — Thi lên cấp, chứng nhận, hồ sơ năng lực (02/11/2026, dev + production)**: `docs/plans/2026-11-02-giai-doan-3-3-thi-cap-chung-nhan-ho-so.md` (mục 9 "Kết quả"). Chế độ thi ở server (đồng hồ theo giờ server, tự nộp, mỗi thế một lượt), quản lý duyệt → cấp + chứng nhận A4 ngang + lịch sử cấp + 10 sao, giao diện "Thi cấp" / "Phòng thi" / "Con tôi", action `profile` + khối "Tiến bộ" (plugin `dsc-chess` 0.2.3).
- [x] **Giai đoạn 4.1 — Giải đấu (02/11/2026, dev + production)**: `docs/plans/2026-11-02-giai-doan-4-mo-rong.md` (mục 8 "Kết quả"). Schema 6 bảng + plugin `dsc-tournament` 0.3.0 (Thuỵ Sĩ / vòng tròn, xếp hạng, Elo, TRF, giấy khen, trang đăng ký công khai + VietQR), webhook mã `GD` (`dsc-bank-webhook` 0.2.1), `da_thu` kỳ thủ tự tính từ phiếu thu (F10 / F11), W18 / W19, giao diện desktop nhóm "Giải đấu", mobile "Trọng tài", "Giải đấu & Elo" trong khối "Tiến bộ" của phụ huynh. 4.2–4.5 chưa làm.
- [x] **Giai đoạn 4.2 — Hiệu sách & phân phối: XONG 03/11/2026 (dev + production, plugin `dsc-shop` 0.4.0, verify 846 phép đạt, Playwright 8.9 đạt)** — chi tiết mục 11 của kế hoạch 4.2. Bước đầu 4.2-1 + 4.2-2 (02/11/2026): `docs/plans/2026-11-02-giai-doan-4-2-hieu-sach-phan-phoi.md` (mục 11 "Kết quả"). 12 bảng bán hàng & kho, vai trò `dsc_ban_hang`, công thức tồn kho / đơn / công nợ (SQLite + PostgreSQL), workflow F12–F20 + W20–W26, plugin `dsc-shop` 0.1.0 chặn xuất âm; 37 sản phẩm từ danh mục sách, 3 kho, tệp kiểm kê đầu kỳ. Sau đó: cửa hàng công khai, webhook `DH`, Shopee, nhà sách (4.2-3/4.2-4), giao diện nhóm "Bán hàng" + dashboard + mobile "Sách"/"Bán sách" (4.2-5). 4.3–4.5 chưa làm.

## 7. Kiểm chứng kế hoạch

- File tồn tại đúng đường dẫn `docs/plans/`, nội dung tiếng Việt, khớp 4 quyết định đã chốt.
- Mọi plugin được liệt kê ở 2.3 có thật trong `packages/plugins/@nocobase/` (đã đối chiếu danh sách thư mục).
- Không sửa mã nguồn, không commit (trừ khi chủ dự án yêu cầu).
