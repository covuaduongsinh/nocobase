# Prompt bàn giao — Hệ thống Dương Sinh trên NocoBase (phiên 30/10/2026)

> Dán toàn bộ khối dưới đây vào phiên mới. Không chứa mật khẩu / API key; mọi bí mật nằm trong file ở máy (đường dẫn bên dưới).

---

Bạn tiếp tục dự án **hệ thống quản lý Công ty CP Cờ vua Dương Sinh trên NocoBase**. Trả lời bằng tiếng Việt. Đọc kỹ phần này trước khi làm gì.

## 1. Bối cảnh & tài liệu

- Hồ sơ công ty: `E:\Dropbox\OBSIDIAN2026\01 - Notes\Gioi-thieu-Cong-ty-CP-Co-vua-Duong-Sinh-thang 9-2026-CapNhat.md`.
- Kế hoạch (mọi kế hoạch để trong `D:\code\nocobase\docs\plans\`):
  - `2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md` — chiến lược, lộ trình 4 giai đoạn, **mục "Tiến độ" là checklist cập nhật nhất**.
  - `2026-10-30-giai-doan-1-van-hanh-clb-dac-ta.md` — đặc tả giai đoạn 1 (mô hình dữ liệu, workflow W0–W6, phân quyền).
  - `2026-11-01-nocobase-studio-dac-ta.md`, `…-dac-ta-demo-api.md` (giải mã demo qua OpenAPI/CLI manifest), `…-phu-luc-api-cli.md` (653 route + 389 lệnh), `…-ke-hoach-phat-trien.md`, `2026-11-01-prompt-trien-khai-nocobase-studio.md` — đặc tả, kế hoạch và prompt dựng "Dương Sinh Studio" (bản tự dựng của NocoBase Studio trên NocoBase 3, repo riêng `duongsinh-studio`; chưa bắt đầu, chờ chốt 6 quyết định).
- Mã riêng Dương Sinh: repo git cục bộ **`D:\code\duongsinh-nocobase`** (chưa có remote). Đọc `README.md` trước. `D:\code\nocobase` là mã nguồn NocoBase 2.2.36 — **không sửa lõi**.

## 2. Hiện trạng

- **Production:** https://nocobase.dsc.edu.vn (desktop `/admin`, giáo viên trên điện thoại `/v/mobile`). VPS 217.15.160.118 (Ubuntu 24.04, SSH `root`, key `~/.ssh/id_ed25519`), quản lý bằng Dokploy https://dokploy.dsc.edu.vn — project `nocobase-dsc`, compose `nocobase` (appName `nocobase-dsc-jyvcqc`): NocoBase 2.2.36 + PostgreSQL 16, HTTPS Let's Encrypt.
- **Đã dựng xong (script tái chạy an toàn, `node setup/run.mjs`):** 13 collection `dsc_*`, danh mục (3 cơ sở, 6 cấp Tốt→Vua, 3 gói 10/20/40 buổi), 4 vai trò (`dsc_quan_ly`, `dsc_giao_vien`, `dsc_ke_toan`, `dsc_tu_van`), 7 workflow W0–W6, 13 trang desktop + 2 trang mobile, quyền menu theo vai trò, ngôn ngữ mặc định vi-VN.
- **Dữ liệu thật đã nhập từ Lark Base** (`https://duongsinhedu.sg.larksuite.com/base/V7DHb5fvQaqnrGsH6fFlysuJgme`, script `setup/import-lark.mjs`, khớp theo `lark_id`): 179 học viên, 48 phụ huynh, 7 lớp, 349 đăng ký gói (số buổi còn lại khớp Lark 313/313 nhờ `so_buoi_da_hoc_truoc`). Không nhập CCCD. Không tạo phiếu thu (Lark không lưu số tiền).
- **Nhân sự / tài khoản:** `truc.nm` (Nguyễn Minh Trúc — Giáo viên + Quản lý), `an.pq` (Phạm Quốc An — Giáo viên; "Andy Phạm" trên Lark), `phuonganh.nn`, `ha.pn` (Giáo viên). Phùng Đức Việt: giáo viên cũ, hồ sơ "Đã nghỉ", không tài khoản.
- **Tài khoản của chủ dự án:** `duongsinhchess` (tự đăng ký) — vai trò `dsc_quan_ly` (chỉ Quản lý, theo yêu cầu). Cấu hình hệ thống dùng tài khoản root. Tự đăng ký tài khoản **vẫn bật** (chủ dự án chọn giữ nguyên; người tự đăng ký nhận vai trò `member` = chỉ xem bản ghi của mình).
- **Phân công lớp:** Cô Trúc — VP2-CA1, VP2-CA2, VP6-CA1, VP6-CA2 (Vĩnh Phúc); Thầy An — KL3, KL5, KL7 (Kim Liên).
- **Sao lưu:** cron VPS `30 20 * * *` (= 02:30 giờ VN) chạy `/usr/local/bin/nocobase-dsc-backup.sh` → `/srv/backups/nocobase-dsc/` (giữ 14 ngày) + Google Drive `gdrive-dsc:NocoBase-DSC-backups` (giữ 30 ngày). Khôi phục thử: `/usr/local/bin/nocobase-dsc-restore-test.sh`.

## 3. Quyết định đã chốt với chủ dự án (không hỏi lại)

- NocoBase là nền tảng duy nhất; ưu tiên plugin mã nguồn mở; mã riêng nằm ở repo riêng, không fork lõi.
- **Chỉ "Có mặt" mới trừ buổi** (học bù / vắng có phép / vắng không phép không trừ).
- **Giữ 6 cấp** Tốt→Vua; quy đổi Lark: NM1/NM2 → Tốt, CB/TC <quân> → <quân>.
- Nhập toàn bộ 179 học viên (kể cả đã nghỉ).
- Phân công giáo viên làm trên NocoBase là chuẩn; nhập lại từ Lark không ghi đè giáo viên của lớp đã có.

## 4. Bí mật (chỉ trên máy, không commit, không in ra chat)

- `D:\code\duongsinh-nocobase\deploy\dokploy\.env.secret` — `APP_KEY`, mật khẩu DB, tài khoản root production (`INIT_ROOT_EMAIL` / `INIT_ROOT_PASSWORD`).
- `D:\code\duongsinh-nocobase\deploy\dokploy\staff-accounts.secret` — mật khẩu tạm 4 nhân sự (cột 2 là URL; dòng `localhost` là bản dev).
- Dokploy API key: **chủ dự án cần cấp lại** (key cũ đã lộ trong chat, đã khuyên đổi). Dùng qua biến `DOKPLOY_API_KEY`.
- Lark: `lark-cli` đã đăng nhập (`--as user`, quyền base + search); nếu hết hạn, dùng `lark-cli auth login --domain base --scope "search:docs:read" --recommend --no-wait --json` và đưa link cho chủ dự án.

## 5. Cách chạy (bẫy môi trường Windows)

- Lệnh với production: `cd D:\code\duongsinh-nocobase`, rồi
  `NB_URL=https://nocobase.dsc.edu.vn NB_ACCOUNT=<INIT_ROOT_EMAIL> NB_PASSWORD=<INIT_ROOT_PASSWORD> node setup/<script>.mjs` (đọc 2 giá trị từ `.env.secret` bằng `grep ... | cut -d= -f2-`, đừng in ra).
- Script: `run.mjs [settings|collections|seed|roles|workflows|ui|all]`, `verify.mjs` (kiểm thử đầu-cuối, phải "tất cả đạt"), `recalc.mjs` (đối soát số buổi/đã thu/sao — chạy sau mọi lần nhập hàng loạt), `staff.mjs`, `import-lark.mjs <thư mục ndjson> [--dry-run]`, `demo.mjs load|today|clear` (**chỉ dùng ở dev**).
- Deploy lại: `DOKPLOY_API_KEY=... node deploy/dokploy/deploy.mjs deploy|status`.
- **Dev local** (`D:\code\nocobase`, SQLite, dữ liệu mẫu `[DEMO]`): Node mặc định v25 làm NocoBase lỗi → dùng Node 22 và đặt PATH trực tiếp, **không dùng `fnm exec`** (lặp khoá PATH trên Windows), **không dùng `yarn`** (máy có yarn 4, repo cần yarn 1):
  ```powershell
  $env:Path = "$env:APPDATA\fnm\node-versions\v22.23.4\installation;D:\code\nocobase\node_modules\.bin;$env:Path"
  nocobase-v1 dev --rsbuild   # UI :13000, API :13001
  ```
  SQLite dev hay `SQLITE_BUSY` khi nhiều workflow chạy → chạy `recalc.mjs`. Không có mật khẩu PostgreSQL cục bộ (chủ dự án không có).
- **Luôn thử ở dev trước, rồi mới production.** Dữ liệu thật thử ở dev xong phải xoá (`lark_id` không rỗng). Xoá file xuất Lark sau khi nhập.
- Git Bash trên Windows làm hỏng UTF-8 khi truyền JSON tiếng Việt qua tham số → dựng payload trong Node, không dùng `curl -d '...'` có dấu.
- Commit trong `duongsinh-nocobase` theo Conventional Commits (tiếng Anh), kèm dòng `Co-Authored-By`. Kiểm tra không commit file `*.secret`, `*.ndjson`.

## 6. Bẫy NocoBase đã gặp

- `flowSurfaces:applyBlueprint` (dựng trang): collection > 10 trường cần `defaults.collections.<c>.fieldGroups` phủ **mọi** trường (kể cả `createdAt`, `updatedAt`, `lark_id`). Thêm trường mới vào schema → nhớ thêm vào `FIELD_GROUPS` / `SYSTEM_FIELDS` trong `setup/ui/pages.mjs`.
- NocoBase 2.2.36 báo nhầm `missing-default-field-groups` khi popup có 2 bảng liên quan hoặc popup Xem có thêm trường mới → đang né: popup Học viên chỉ có bảng "Gói đã đăng ký"; popup Xem đăng ký gói không hiện `so_buoi_da_hoc_truoc`; bảng liên quan khai báo popup Xem/Sửa tường minh.
- Trang mobile nằm ở client v2: `/v/mobile` (không phải `/mobile`); route mobile lưu chung `desktopRoutes`. Trên mobile nút chỉ hiện icon.
- Workflow đổi định nghĩa → `run.mjs workflows` tự tạo phiên bản mới (dấu vân tay ở `description`). W0 chạy đồng bộ.
- VPS đang tải nặng (lúc deploy chỉ còn ~1,5 GB RAM trống; NocoBase ~700 MB).

## 7. Việc còn lại (ưu tiên từ trên xuống)

1. ~~Xếp học viên vào 7 lớp~~ — **ĐÃ LÀM** (suy từ Lark "2026 Nhận xét mỗi buổi" `tbl9TTf0m2EhWV3g`, 20/8–15/10/2026: lớp đi nhiều nhất + mọi lớp ≥2 buổi; một em có thể nhiều lớp — chủ dự án xác nhận). KL3 20, KL5 28, KL7 24, VP2-CA1 9, VP2-CA2 12, VP6-CA1 16, VP6-CA2 10 em. Đã xử lý theo chủ dự án: gộp 4 cặp trùng (`setup/merge-students.mjs` + `setup/data/student-merges.json`; Lê Minh Khôi là 2 em khác nhau, giữ cả hai); cập nhật trạng thái 4 em. Hiện: 175 học viên (87 đang học — tất cả đã có lớp, 26 bảo lưu, 51 nghỉ, 11 học thử). **Từ nay NocoBase là nguồn chuẩn**: nhập lại từ Lark chỉ điền trường trống, bỏ qua hồ sơ đã gộp, không xếp em "nghỉ" vào lớp. Chạy lại: xuất bảng nhận xét (nhớ phân trang `--offset 2000`), ghép thành `nhanxet.ndjson`, `node setup/import-lark.mjs <thư mục>`.
   1b. **Đã làm thêm (30/10 tối):** chủ dự án quyết định **mọi nhân viên, giáo viên thấy tất cả các lớp** (không bật giới hạn "chỉ lớp mình" — bỏ việc 4 bên dưới). Đã nhập lịch sử nhận xét 2024–2026 (625 buổi "Đã dạy", 5.635 lượt điểm danh "Có mặt" kèm nhận xét, KHÔNG gắn gói) và 7 lead. File rà soát (có tên học viên, không commit) ở `D:\code\duongsinh-nocobase\reports\`: `1-dang-hoc-khong-co-goi-con-buoi.csv` (44/87 em, 15 em chưa từng có gói), `2-hoc-vien-thieu-du-lieu.csv` (124 em đang theo học: 52 thiếu ngày sinh, 92 thiếu phụ huynh, 122 thiếu cấp độ). Hướng dẫn client_id rclone: `deploy/dokploy/HUONG-DAN-RCLONE-CLIENT-ID.md` (chờ chủ dự án tạo trên Google Cloud).
   1c. **Đã làm thêm (Deal HP + Tổng Deal):** 4.459 buổi lịch sử đã gắn vào đúng kỳ học phí (số buổi còn lại khớp Lark 309/313; 4 kỳ lệch 1–4 buổi do Lark đếm thiếu: DK00046, DK00141, DK00171, DK00223); 1.176 lượt lịch sử không thuộc kỳ nào trên Lark. 270 phiếu thu từ số tiền thật (tổng 511.134.000đ: 2024 231,8tr, 2025 220,3tr, 2026 59tr — 2026 có vẻ thiếu, cần chủ dự án xác nhận Lark còn ghi lần nộp ở đâu khác); 195 phiếu gắn kỳ, 30 chỉ gắn học viên, 45 chỉ có tên phụ huynh (gắn tay). Học phí kỳ = số tiền thật; kỳ không rõ tiền để trống.
   1d. **Sách (30/10):** danh mục 37 sách (`dsc_sach`, trang Danh mục → Sách) từ Lark `tbl324dOlvbf3BUu`; học viên có `sach_dang_hoc` / `sach_da_hoc` (m2m), điểm danh có `sach` (m2m). Gắn sách cho 3.538 buổi lịch sử; suy sách đang học cho 71/87 em đang học bằng `setup/infer-books.mjs` (buổi gần nhất trong 60 ngày). Vai trò giáo viên được thêm quyền **sửa học viên** để cập nhật sách. Gói học phí: popup "Xem" có bảng "Các buổi đã học trong gói"; phiếu thu ở nút riêng "Phiếu thu".
   1e. **Tiền sách (30/10):** `dsc_phat_sach` 338 lần phát sách (Lark "Từng lần phát sách", 2023-07 → 2025-09, tổng 17,47tr; 5 dòng thiếu sách, 1 thiếu học viên) — trang Học phí → Phát sách; `dsc_phieu_thu.loai_thu` (hoc_phi / tien_sach / khac): 176 phiếu tiền sách 31,44tr (từ Tổng Deal + 1 khoản Deal book chưa trùng; 36 phiếu chỉ có tên phụ huynh). Biểu đồ doanh thu chỉ tính học phí. Chưa tính số dư tiền sách (dữ liệu phát sách thiếu từ 9/2025).
   1f. **31/10:** trang mobile "Phát sách" cho giáo viên + workflow **W7** (đồng bộ) tự điền giá theo danh mục, thành tiền = giá − ưu đãi (dùng math.js: trường decimal về dạng chuỗi, `N()` của formula.js trả 0). 81 phiếu thu chưa gắn học viên (70,8tr; tên người nộp là tên chuyển khoản, không ghép tự động được) → `reports/3-phieu-thu-chua-gan-hoc-vien.csv` để gắn tay.
2. **Rà học viên "Đang học" không có gói còn buổi** (đã có file `reports/1-...csv`, chờ chủ dự án xử lý) (thiếu kỳ học phí mới trên Lark hoặc trạng thái cũ) — lập danh sách cho chủ dự án xác nhận (không tự sửa).
3. Bổ sung dữ liệu thiếu cho 127 em đang theo học: 52 thiếu ngày sinh, 96 chưa gắn phụ huynh, gần như chưa có cấp độ; 4 ngày sinh bất thường.
4. Bật giới hạn "giáo viên chỉ thấy lớp mình" (data scope theo `{{$user.id}}` qua `dsc_nhan_su.tai_khoan`), sau khi đã xếp học viên vào lớp.
5. Nhập tiếp từ Lark khi được yêu cầu: nhận xét buổi học 2024–2026 (`tblprrzroE2NBli5`, `tbluhrY4B8i63PAh`, `tbl9TTf0m2EhWV3g`), Lead (`tbl2MF2cmS7CLEkb`).
6. Tạo OAuth client_id riêng cho rclone Google Drive (client_id dùng chung của rclone bị Google ngừng trong 2026) — chủ dự án tự đăng nhập Google.
7. Giai đoạn 2 theo kế hoạch: cổng phụ huynh, Zalo OA, VietQR. **2.1 cổng phụ huynh xong 31/10/2026** (xem mục Tiến độ của kế hoạch tổng và README mục "Cổng phụ huynh"). Bẫy mới:
   - NocoBase 2.2.36 không áp data scope / whitelist cột cho `appends` (kể cả lồng nhau) và route `<collection>/<id>/<quan hệ>` → plugin `@duongsinh/plugin-dsc-guard` (mã ở `duongsinh-nocobase/plugins/`) là **bắt buộc** khi có tài khoản phụ huynh; tắt plugin thì `verify.mjs` mục 7 báo trượt.
   - Build plugin: `nocobase-v1 build` gọi `yarn` (máy có yarn 4) → dùng `node plugins/build.mjs <tên>` (gọi thẳng `tsup` + `nocobase-build`). Dev cần junction `node_modules\@duongsinh\<plugin>`.
   - Quyền menu cấp theo **tên** mục gốc: trang mobile trùng tên nhóm desktop (vd "Học phí") sẽ cấp nhầm → `menu-access.mjs` giờ báo lỗi khi trùng.
   - Trình duyệt còn lưu vai trò `root` của phiên trước → đăng nhập phụ huynh bị đá về màn đăng nhập (`ROLE_NOT_FOUND_FOR_USER`); xoá localStorage hoặc dùng trình duyệt khác khi thử.
   - Dev: tài khoản phụ huynh thử `0912000001` (dữ liệu [DEMO]), mật khẩu trong `parent-accounts.secret` dòng localhost.
   - NocoBase mở `workflows:trigger` (workflow "nút bấm" toàn cục) cho mọi tài khoản đăng nhập → plugin dsc-guard 0.1.1 chỉ cho `root`/`admin`/`dsc_quan_ly` (commit `6c474ac`). Workflow nút bấm mới chạy được từ vai trò nào khác thì phải thêm vào `WORKFLOW_TRIGGER_ROLES` trong `guard.ts`.
   - Cập nhật plugin đã cài: `install.mjs` gửi `pm:update?filterByTk=<tên gói>` (mọi action update của NocoBase bắt buộc filterByTk).
   - Production: 22 tài khoản phụ huynh thật đã cấp 31/10/2026 (mật khẩu tạm trong `parent-accounts.secret`). **Đừng đăng nhập thay phụ huynh để thử** — làm hỏng số liệu kích hoạt (`users.lastSignInAt`); kiểm tra bằng tài khoản quản trị hoặc `verify.mjs parents`.
   - 2.2 VietQR: tài khoản nhận học phí VPBank (BIN 970432) số 168686838, Công ty CP Cờ vua Dương Sinh (MST 0108526687), tên chủ TK trên ngân hàng: **DUONG SINH CHESS JSC** (chủ dự án xác nhận 31/10/2026).
   - **2.3 tuyển sinh và 2.4 Zalo ZNS xong (31/10/2026)**, commit `25ded42` và `f69b6d5`/`312cfec`. ZNS ở chế độ `tat` trên production; chờ chủ dự án (App ID/secret vào Dokploy, 3 mẫu ZNS, refresh token, SĐT thử) — xem `duongsinh-nocobase/docs/zns-mau-de-xuat.md`, `node setup/zalo.mjs status`. Bẫy: id snowflake → workflow không INSERT thô; endpoint Zalo chưa đối chiếu tài liệu chính thức. Bước kế: 2.5 webhook SePay.
   - **2.5 Webhook SePay xong phần mã (01/11/2026)**, commit `b701122`: endpoint production trả 503 tới khi chủ dự án đăng ký SePay và đặt `DSC_SEPAY_API_KEY` (`duongsinh-nocobase/docs/huong-dan-sepay.md`). Chưa thử webhook trên PostgreSQL (cần khoá thử). Hết giai đoạn 2 về mặt mã.
   - **BÀI HỌC 01/11/2026 (sự cố giao diện web production ~1 ngày):** plugin chỉ-server (`dsc-zalo`, `dsc-bank-webhook`) thiếu `client.js`/`server.js` ở gốc → không có `dist/client` → trình duyệt 404 `/static/plugins/@duongsinh/<plugin>/dist/client/index.js` → "Script error" → web không render, trong khi API và mọi `verify.mjs` vẫn đạt. Quy tắc: plugin mới phải chép `client.js`, `server.js`, `src/client/index.tsx` từ `plugin-dsc-guard`; sau mỗi lần cài/cập nhật plugin chạy `node setup/verify.mjs plugins` VÀ `node setup/web-check.mjs <url>` (trình duyệt thật). Sửa ở commit `ba81131`; kiểm tra tự động `web-check.mjs`.
   - **3.1-0 bàn cờ đạt (01/11/2026):** chess.js 1.4.0 (esm.sh) + cm-chessboard 8.15.4 (jsDelivr) qua `ctx.importAsync` trong jsBlock; kéo thả + chạm-chạm trên mobile; tải 0,6–1,8s lần đầu. Bản nháp `setup/ui/chess-board.mjs` (`4b9b5a9`). Validator chặn alias `ctx.element` và `ctx.loadCSS`: render bằng `ctx.render(React.createElement(...))`.

   - **SỰ CỐ 01/11/2026 (cửa sổ sai quyền, không có phụ huynh thật bị ảnh hưởng):** `plugin-acl/server.ts` (`fields.afterCreate`) tự thêm MỌI trường mới — kể cả trường ngược của quan hệ vừa tạo — vào whitelist cột `view` của mọi vai trò, gồm `dsc_phu_huynh`. Giữa `run.mjs collections` và `roles`, NocoBase nạp kèm `lam_bai` (+ `createdById`) vào `dsc_hoc_vien:list` mà guard 0.1.3 không cắt. Log request production cho thấy 0 request của 22 tài khoản phụ huynh thật (31/10–01/11). Sửa ở guard 0.1.4: danh sách cột đã duyệt sinh từ `apply.mjs` (`node setup/acl/gen-parent-columns.mjs`), đóng mặc định, cắt body theo danh sách; `run.mjs collections` luôn kéo theo `roles`. **Thêm trường cho phụ huynh xem = sửa `apply.mjs` + chạy `gen-parent-columns.mjs` + build lại guard; trường admin thêm trên giao diện bị guard chặn — chủ ý.** Có phép hồi quy trong `parent-oracle.mjs` (`checkNewFieldHidden`).

   - **GIAI ĐOẠN 3 XONG (01/11/2026)** — LMS 6 cấp, xem `docs/plans/2026-11-01-giai-doan-3-lms-6-cap.md` mục 12. Bẫy mới: (a) vai trò nhân viên PHẢI có `fields` tường minh cho view/export/create/update (action không có `fields` = whitelist rỗng, chỉ cột hệ thống; root không bị nên test bằng root không thấy) — `apply.mjs` lo, `verify.mjs staff` kiểm; (b) `Repository.find/findOne` có `appends` + `transaction` văng TypeError (cloneDeep Transaction) → plugin đọc không truyền transaction; (c) dev SQLite chập chờn `SQLITE_BUSY` khi nhiều workflow chạy: verify dev thử lại / chạy từng mục (`verify.mjs staff|parents|lms`), PostgreSQL production ổn; dev không tự nạp lại plugin sau build → kill tiến trình `nocobase-v1 dev` và chạy lại; (d) validator RunJS chặn `ctx.request` tới collection (dùng action server như `dscChess:children`); (e) bài học phụ huynh lọc theo cấp của con: bài chưa gắn cấp phụ huynh không thấy. Việc của chủ dự án: giáo viên duyệt 20 thế cờ mẫu + soạn học liệu rồi xuất bản; chọn lớp thật để giao bài.
   - **3.3 / 3.4 XONG (02/11/2026)** — thi lên cấp + chứng nhận + hồ sơ năng lực, xem `docs/plans/2026-11-02-giai-doan-3-3-thi-cap-chung-nhan-ho-so.md` mục 9. Plugin `dsc-chess` 0.2.3. Bẫy mới: (a) validator RunJS cấm ghi DOM trực tiếp (`document.createElement`) và biến global lạ → dùng `ctx.render` + React, tham chiếu qua ref; mã khối nằm trong template string nên không dùng backtick / `${` / regex; (b) `crudPage` chưa có tab trong popup → khối "Tiến bộ" là khối thêm vào popup; (c) bản dev SQLite treo khi chạy `verify.mjs exam` → chạy phép kiểm tích hợp trên production sau khi sao lưu; (d) `verify.mjs` mục 13 dùng `setup/exam-check.mjs`, dọn xong đặt lại số chứng nhận bằng `setup/sequence.mjs`. Việc của chủ dự án: soạn đề thi từng cấp, duyệt mẫu chứng nhận (chữ ký, con dấu), bổ sung cấp ban đầu cho ~122 em.
   - **4.1 GIẢI ĐẤU XONG (02/11/2026)** — xem `docs/plans/2026-11-02-giai-doan-4-mo-rong.md` mục 8. Bẫy mới: (a) action `acl.allow(..., 'public')` của NocoBase KHÔNG nhận diện người đăng nhập → bản nội bộ phải là action riêng (`refereeStandings`…); (b) validator flowSurfaces từ chối popup có hơn 3 khối cạnh chi tiết học viên (nên "Giải đấu & Elo" nằm trong khối "Tiến bộ") và đòi `FIELD_GROUPS` cho collection có > ~10 trường popup (thêm trường mới vào `pages.mjs`); (c) `dsc_ky_thu.da_thu` / trạng thái xác nhận là TRƯỜNG TỰ TÍNH (F10), webhook và kế toán chỉ tạo phiếu thu `le_phi_giai` gắn kỳ thủ; (d) đăng ký công khai giới hạn 5 lượt / 10 phút / IP nên verify / Playwright chạy lặp phải chờ cửa sổ; kỳ thủ đăng ký web không nằm trong `create()` của verify nên check tự dọn; (e) dev: tiến trình rsbuild / tsx cũ có thể sót khi khởi động lại → port 13001 bận, lỗi 431; kill mọi node có `rsbuild|cli-v1|tsx/dist` rồi chạy lại. Việc của chủ dự án: điều lệ giải, mẫu ZNS "đăng ký giải", duyệt staging.
   - **4.2 HIỆU SÁCH & PHÂN PHỐI XONG (03/11/2026)** — plugin `dsc-shop` 0.4.0, xem `docs/plans/2026-11-02-giai-doan-4-2-hieu-sach-phan-phoi.md` mục 11. Nhóm desktop "Bán hàng" (10 trang), mobile "Sách" (Phát sách + Bán sách) và "Bán sách" cho `dsc_ban_hang`; trang mobile "Phát sách" cũ đã xoá. Bẫy mới: (a) áp UI production dùng `DSC_PAGES="..."` (cả bộ > 10 phút); (b) sau `verify.mjs` đầy đủ phải chờ ~11 phút mới chạy `ui-shots-ban-hang.mjs` (hạn mức 5 đơn / 10 phút / IP); (c) `cleanOrphanIssueSlips` phải chạy SAU khi xoá lần phát sách thử (đã sửa trong `verify.mjs`); (d) sau dọn tồn: `recalc.mjs --apply`. Việc của chủ dự án: kiểm kê đầu kỳ (`reports/5-kiem-ke-dau-ky.csv`), file Shopee thật để xác minh tên cột. Bước kế: 4.3–4.5 (chưa có kế hoạch chi tiết).

Bắt đầu bằng việc đọc `README.md` của `D:\code\duongsinh-nocobase` và mục "Tiến độ" trong kế hoạch chiến lược, kiểm tra `git status` và `git log --oneline | head`, rồi hỏi chủ dự án muốn làm việc nào trước (mặc định: việc 1 nếu đã có danh sách lớp).
