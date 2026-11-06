# Kế hoạch chuyển hệ thống Dương Sinh (production) từ NocoBase 2 sang NocoBase 3 — 03/11/2026

> Trạng thái: **đã duyệt 03/11/2026.** Chốt: không chờ GA; chuyển dữ liệu đang dùng (công cụ `migrate/` có đối soát, chạy
> thử trên bản sao đã che dữ liệu cá nhân); phần giải đấu làm tiếp thẳng trên v3 (không hoàn thiện trên v2); repo mới
> `duongsinh-app`; bản mẫu là Học viên – Lớp – Buổi học – Điểm danh; **ưu tiên v3 trước S6** (S6.1 đã xong vẫn đưa lên
> production, S6.2+ tạm dừng). Thứ tự: S6.1 lên production → A → B → C → D.
> **Cập nhật chiều 03/11/2026:** production `nocobase.dsc.edu.vn` đã được chủ dự án (qua Antigravity) chuyển sang một app
> NocoBase 3 dựng thẳng từ monorepo `D:\code\nocobase` (`v3-main` + 7 file lõi sửa tay, SQLite, bảng `dsc_*` chép thô từ bản
> sao lưu v2 bằng `scripts/migrate_v2_to_sqlite.py`); volume Postgres v2 và bản sao lưu `backup_v2_before_v3_cutover_.sql.gz`
> còn nguyên. Chủ dự án chọn: **giữ bản v3 hiện tại trên production** trong lúc xây `duongsinh-app` (repo riêng, create-app,
> Postgres, làm qua Studio) rồi thay dần; 11 commit chỉ có trên máy chủ dự án đã quét bí mật và đẩy lên nhánh
> `v2/local-2026-11-03` của `duongsinh-nocobase` (`cff46e1`). Kiểm kê giai đoạn B: `docs/plans/v3/kiem-ke-v2.md`.
> **Tăng tốc (03/11/2026 tối, chủ dự án: "đẩy nhanh, tự động triển khai toàn bộ các công việc"):** bật thêm runner máy dev
> (`may-dev-duongsinh-prod`, 2 lượt) cạnh runner VPS (1 lượt); gắn `duongsinh-nocobase` làm repo phụ (chỉ đọc) của project
> "Dương Sinh App" để agent đọc nhánh `v2/local-2026-11-03`; mở song song DS-10…DS-14 (chuyển logic thuần: giải đấu, chấm
> cờ, tồn kho/Shopee, webhook ngân hàng/VietQR, công thức). Điều phối viên merge khi đủ điều kiện (CI xanh, review Opus
> không còn Must fix, có báo cáo bàn giao, đã tự đọc PR) và mở tiếp các mảng của giai đoạn D theo thứ tự kiểm kê. Review
> tự động do Claude Opus 5.5 (Codex tắt). Cắt chuyển production (E) và mọi thao tác ghi lên dữ liệu thật vẫn báo/hỏi chủ dự
> án trước.
> Giai đoạn A làm bằng **bản clone riêng** `D:\code\nocobase-v2` (không thêm worktree vào `D:\code\nocobase`, thư mục đó chỉ
> đọc). Chưa thay đổi gì ở production, repo hay VPS.
> Lý do lập: chủ dự án muốn `nocobase.dsc.edu.vn` chạy NocoBase 3 (đã chuyển thư mục nguồn tham khảo `D:\code\nocobase`
> sang nhánh `v3-main` ngày 03/11/2026).
> Liên quan: `2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md` (mô hình dữ liệu, plugin dùng), các kế hoạch giai đoạn
> 1–4 trong `docs/plans/`, Dương Sinh Studio (`2026-11-01-nocobase-studio-*.md`, đã chạy trên NocoBase 3).

## 1. Sự thật hiện tại (kiểm 03/11/2026, chỉ đọc)

| Mục                              | Hiện trạng                                                                                                                                                                                                                                 |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Production `nocobase.dsc.edu.vn` | NocoBase **2.2.36** (`deploy/dokploy/docker-compose.yml`, image `nocobase/nocobase:2.2.36-full`), Postgres 16, dữ liệu thật học viên/phụ huynh/học phí                                                                                     |
| Mã riêng                         | Repo `duongsinh-nocobase`: plugin `@duongsinh/*` cho client v2 (FlowModel) + script `setup/*.mjs` dựng schema, ACL (cách ly dữ liệu phụ huynh), workflow (công thức, giải đấu, Zalo, SePay), ~22 trang giao diện + mobile + form công khai |
| NocoBase 2 mới nhất              | 2.2.37 (31/10/2026), vẫn được phát hành tiếp                                                                                                                                                                                               |
| NocoBase 3                       | **Beta**: nhánh `v3-main` chưa có số phiên bản (`0.0.0`), công cụ tạo app `0.1.0-beta.34`, gói lõi `app-server 4.0.0-beta.0`; Docker Hub chưa có image v3; tài liệu nhiều trang còn "Being written"                                        |
| Đường nâng cấp v2 → v3           | **Không có.** v3 là nền tảng viết lại (ứng dụng viết bằng mã: collection, trang React/shadcn, workflow DSL, quyền bằng permission set), không đọc được cấu hình UI/workflow/ACL của v2. Không có công cụ chuyển dữ liệu chính thức         |

Hệ quả: "nâng cấp" thực chất là **dựng lại ứng dụng Dương Sinh trên NocoBase 3 rồi chuyển dữ liệu sang**, không phải đổi phiên
bản image.

## 2. Đối chiếu năng lực v2 đang dùng ↔ v3 hiện có

| Nhu cầu (đang chạy trên v2)                                          | v3 hôm nay                                                               | Đánh giá                                               |
| -------------------------------------------------------------------- | ------------------------------------------------------------------------ | ------------------------------------------------------ |
| Collection, quan hệ, trường công thức/sequence                       | `@nocobase/db` + collection bằng mã, migration                           | Có, phải viết lại bằng mã                              |
| Vai trò + cách ly dữ liệu phụ huynh (data scope, `plugin-dsc-guard`) | `authorization`: permission sets, data scopes, restriction/sharing rules | Có khung, phải viết lại và kiểm kỹ (rủi ro lộ dữ liệu) |
| Workflow (công thức tự tính, giải đấu, nhắc gia hạn)                 | `workflow` (DSL bằng mã, node condition/run/terminate)                   | Có, ít node hơn v2; phần lớn viết thành mã             |
| Thông báo trong app, email                                           | `notification`, `notification-in-app`, `mail`                            | Có                                                     |
| Zalo ZNS, SePay webhook, VietQR                                      | —                                                                        | Viết lại plugin `@duongsinh/*` cho v3                  |
| Form công khai (đăng ký học thử, giải đấu)                           | Chưa thấy trong v3                                                       | **Thiếu** — tự làm trang công khai                     |
| Ứng dụng mobile cho phụ huynh/giáo viên (`/v/mobile`)                | Chưa có plugin mobile; giao diện React responsive tự làm                 | **Thiếu** — tự làm                                     |
| Lịch, kanban, gantt, biểu đồ, import/export Excel, in phiếu          | `template-print` có; còn lại chưa thấy                                   | **Thiếu phần lớn**                                     |
| Đăng nhập phụ huynh bằng SĐT/OTP                                     | `auth` (có cơ chế thêm phương thức đăng nhập)                            | Phải tự viết phương thức                               |
| Bàn cờ, thi cấp, học liệu (`plugin-dsc-chess`)                       | —                                                                        | Viết lại client bằng React (logic chấm dùng lại được)  |
| Nhật ký, sao lưu trong app                                           | Chưa thấy                                                                | Dùng sao lưu Postgres ngoài app                        |

Kết luận đối chiếu: chuyển ngay lúc này nghĩa là **viết lại phần lớn hệ thống** và tự làm nhiều thứ v2 có sẵn, trên một nền
tảng còn beta đổi API hằng tuần.

## 3. Hướng đi (chủ dự án quyết 03/11/2026)

> "Production của v2 tôi chưa làm hoàn chỉnh. Nên không cần chờ v3 có bản chính thức, mà có thể chuyển production với v3
> hiện tại đang có."

Vì hệ thống v2 chưa hoàn chỉnh, **chuyển hướng phát triển sang v3 ngay** thay vì hoàn thiện v2 rồi viết lại:

1. **Dừng phát triển tính năng mới trên v2** (chỉ sửa lỗi chặn vận hành nếu có); mọi tính năng mới làm thẳng trên v3.
2. Dựng ứng dụng v3 `duongsinh-app` qua Dương Sinh Studio, ưu tiên những mảng v2 đang dùng thật.
3. Cắt chuyển `nocobase.dsc.edu.vn` sang v3 **ngay khi bản v3 phủ đủ các mảng đang dùng** (không chờ GA), với các giảm thiểu
   cho việc chạy beta ở mục 7: ghim phiên bản, nâng có kiểm soát qua staging, sao lưu trước mỗi lần nâng.
4. v2 giữ chạy tới lúc cắt chuyển, sau đó chỉ đọc một thời gian rồi tắt.

## 4. Các giai đoạn

### A — Giữ môi trường v2 làm việc được (ngay, ~0,5 ngày)

> **Kết quả 03/11/2026 (thu gọn vì v2 đã ngừng phát triển):** `plugins/build.mjs` đã đọc nguồn NocoBase từ biến
> `NOCOBASE_SRC` (mặc định `D:/code/nocobase`). Không dựng bản clone v2 cục bộ (tốn `yarn install` vài GB cho mã sắp bỏ).
> Khi cần sửa lỗi gấp trên v2: build bằng CI (`ci/build-plugins.mjs` của PR #1 dựng từ `@nocobase/build@2.2.36` trên npm),
> hoặc clone tag `v2.2.36` vào `D:\code\nocobase-v2` rồi chạy `NOCOBASE_SRC=D:/code/nocobase-v2 node plugins/build.mjs <tên>`.
> Phần dưới là phương án đầy đủ ban đầu, giữ để tham khảo.

- Thư mục `D:\code\nocobase` đã sang v3 nên `plugins/build.mjs` của `duongsinh-nocobase` (cần nguồn NocoBase 2) sẽ hỏng.
  Tạo worktree `D:\code\nocobase-v2` ở tag `v2.2.36` (sau đó `v2.2.37`), cho `build.mjs` đọc đường dẫn từ biến
  `NOCOBASE_V2_SRC` (mặc định thư mục mới), cập nhật README/AGENTS.md.
- Kiểm: build lại các plugin `@duongsinh/*`, so danh sách file `.tgz` với bản cũ; CI PR #1 vẫn xanh.

### B — Kiểm kê hệ thống v2 (~2 ngày)

- Liệt kê đầy đủ, máy đọc được (`docs/v3/kiem-ke-v2.md` + JSON): mọi collection/trường/quan hệ (đọc từ `setup/`), số bản
  ghi mỗi bảng (đếm trên **bản sao** hoặc theo truy vấn đếm do chủ dự án chạy), vai trò và quy tắc ACL, workflow và
  trigger, trang/block, form công khai, tích hợp ngoài (Lark, Zalo, SePay), plugin `@duongsinh/*` và chức năng của từng cái.
- Đánh dấu mỗi mục: dùng thường xuyên / ít dùng / bỏ được. Chủ dự án duyệt danh sách (thu hẹp phạm vi là đòn bẩy lớn nhất).

### C — Bản mẫu v3 cho một mảng (~1–2 tuần, làm qua Studio)

> **Đã làm 03/11/2026 (khởi động):**
>
> - Repo private `covuaduongsinh/duongsinh-app` (WSL `~/code/duongsinh-app`), tạo từ `@nocobase/app-template-default`
>   1.0.0-beta.86 (cùng bản Studio) bằng create-app 0.1.0-beta.34. Commit `373c3a9` (scaffold), `466b281` (vi-VN mặc định,
>   nhận diện Dương Sinh, override Rollup wasm + nodeOptions 4 GB, driver `@nocobase/db-postgres`), `4a44b02` (luật agent
>   `.agents/rules/duongsinh-app.md`, CI `.github/workflows/ci.yml`, README, `docs/kien-truc.md`, `docs/chuyen-du-lieu.md`).
>   Trên máy: lint, typecheck, 86 file / 532 test, build đều xanh; dev cổng 13200.
> - Studio production: dự án **"Dương Sinh App"** (`395715805184000`); epic **DS-3** và các công việc **DS-4** (danh mục nền
>   - vai trò) → **DS-5** (học viên + phụ huynh, mã HV) → **DS-6** (lớp + buổi học) → **DS-7** (điểm danh) → **DS-8** (test
>     cách ly phụ huynh) và **DS-9** (công cụ `migrate/` trên dump giả), đều ở Chờ làm, nối "bị chặn bởi" theo thứ tự.
> - **Chờ chủ dự án:** thêm `duongsinh-app` vào cài đặt GitHub App (Settings → Git → thẻ GitHub App → "Install on
>   repositories" → chọn thêm `duongsinh-app` → Save), rồi liên kết repo ở thẻ Repositories và gắn vào mục Code của dự án
>   "Dương Sinh App". Trước bước này Studio trả `GIT_REPOSITORY_UNREACHABLE` (404).

- Tạo repo mới `covuaduongsinh/duongsinh-app` (NocoBase 3, theo skill `nocobase-create-app`), gắn vào Studio như project
  mới; agent làm theo issue như với sandbox.
- Mảng mẫu đề xuất: **Học viên + Lớp + Buổi học + Điểm danh + quyền giáo viên** (lõi vận hành CLB, đủ để thử data scope).
- Dữ liệu chỉ là dữ liệu mẫu sinh ra; chạy preview v3 (in-process, `app-host`) như demo NocoBase Studio.
- Đầu ra: đánh giá thật thời gian/công sức viết lại 1 mảng → ước lượng toàn bộ; danh sách thiếu hụt v3 cụ thể.
- **Cổng:** chủ dự án xem bản mẫu, chốt cách làm (mẫu mã, giao diện, quyền) để nhân ra các mảng khác.

### D — Viết lại toàn bộ trên v3 + công cụ chuyển dữ liệu (ước tính sau C; nhiều tháng)

- Viết lại từng mảng theo thứ tự ưu tiên của kiểm kê (vận hành CLB → tài chính/phiếu thu → cổng phụ huynh → LMS/thi cấp →
  giải đấu → mở rộng).
- Công cụ chuyển dữ liệu `migrate/` (đọc Postgres v2 → ghi schema v3), chạy lặp lại được, có báo cáo đối soát (số bản ghi,
  tổng tiền phiếu thu, số buổi còn lại theo học viên…).
- Chạy thử chuyển dữ liệu trên **bản sao production đã che dữ liệu cá nhân** (tên, SĐT, email thay bằng giả) trong môi
  trường tách biệt; bản sao thật chỉ dùng ở lần tổng duyệt cuối, trên máy chủ dự án kiểm soát.
- Staging v3 song song staging v2; người dùng thật (giáo viên, kế toán) thử theo kịch bản.

### E — Cắt chuyển (khi đạt mọi cổng)

- Điều kiện: bản v3 phủ đủ các mảng đang dùng (theo kiểm kê B); nghiệm thu song song đạt; tổng duyệt chuyển dữ liệu đạt đối
  soát 100 %; có kế hoạch quay lui đã thử. **Không chờ NocoBase 3 GA** (quyết định 03/11/2026); ghim đúng phiên bản beta đã
  nghiệm thu.
- Đóng băng ghi trên v2 → sao lưu → chuyển dữ liệu → đối soát → mở v3 ở `nocobase.dsc.edu.vn` (v2 chuyển sang tên miền nội
  bộ chỉ đọc 3 tháng) → theo dõi 2 tuần.
- Quay lui: trỏ lại tên miền về v2 + dữ liệu phát sinh trên v3 được xuất ra để nhập lại (kịch bản viết sẵn).

## 5. Ảnh hưởng tới Dương Sinh Studio

- **Không cần sửa Studio** vì việc đổi nhánh `D:\code\nocobase`: Studio dùng gói npm ghim phiên bản. Việc nâng gói beta định
  kỳ (projects 0.1.0-beta.23 → beta.24, agents 2.0.0-beta.1 → beta.2, …) nằm trong S6.6.
- Studio là công cụ chính để làm giai đoạn C–D (agent viết `duongsinh-app` qua issue, PR, CI, review chéo, preview v3).
- S6.1 (tiếng Việt + nhận diện) đã xong trên dev, **tạm giữ chưa lên production** theo yêu cầu "từ từ" của chủ dự án.

## 6. Quyết định cần chủ dự án chốt

| #   | Câu hỏi                                                         | Khuyến nghị                                                                                                                                 |
| --- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Q1  | Nguyên tắc cắt chuyển                                           | **Đã chốt:** không chờ GA; cắt chuyển khi bản v3 phủ đủ các mảng đang dùng                                                                  |
| Q1b | Dữ liệu thật hiện có trên v2                                    | Chuyển toàn bộ dữ liệu đang dùng (học viên, phụ huynh, lớp, buổi, phiếu thu…) bằng công cụ `migrate/`; mảng chưa dùng thật thì không chuyển |
| Q1c | Phần đang làm dở trên v2 (giải đấu, commit `9c19812` chưa push) | Đưa lên PR để lưu lại, không phát triển tiếp trên v2; làm lại giải đấu trên v3                                                              |
| Q2  | Làm giai đoạn A–C ngay?                                         | Có: A ngay, B–C xen với S6 (S6.1 lên production trước)                                                                                      |
| Q3  | Mảng làm bản mẫu (C)                                            | Học viên – Lớp – Buổi học – Điểm danh – quyền giáo viên                                                                                     |
| Q4  | Repo mới cho app v3                                             | `covuaduongsinh/duongsinh-app` (private), giữ `duongsinh-nocobase` cho v2 tới khi tắt                                                       |
| Q5  | Dữ liệu cho chạy thử chuyển đổi                                 | Bản sao đã che dữ liệu cá nhân; bản thật chỉ ở tổng duyệt cuối, do chủ dự án thực hiện                                                      |
| Q6  | Thứ tự với S6                                                   | S6.1 → giai đoạn A → S6.2–S6.3 → B → C → phần còn lại của S6                                                                                |

## 7. Rủi ro

| Rủi ro                                                            | Giảm thiểu                                                                                                                                                                   |
| ----------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| v3 beta đổi API, công viết lại bị bỏ phí                          | Bản mẫu nhỏ trước (C); ghim phiên bản; theo dõi CHANGELOG; nâng qua staging + đủ test                                                                                        |
| Production chạy trên beta (lỗi nền tảng, migration hỏng khi nâng) | Ghim đúng phiên bản đã nghiệm thu; sao lưu tự động hằng ngày ra ngoài VPS; chỉ nâng sau khi staging qua kịch bản kiểm thử; giữ bản sao lưu + bản phát hành trước để quay lui |
| Thiếu tính năng v2 (form công khai, mobile, lịch, import Excel…)  | Kiểm kê + thu hẹp phạm vi (B); tự làm phần thật sự cần                                                                                                                       |
| Lộ dữ liệu cá nhân khi chạy thử                                   | Chỉ dùng bản sao đã che; tổng duyệt thật trên môi trường chủ dự án kiểm soát                                                                                                 |
| Sai quyền phụ huynh khi viết lại ACL                              | Test tự động cách ly dữ liệu như `verify.mjs` hiện có, chạy trong CI v3                                                                                                      |
| Người dùng phải học giao diện mới                                 | Nghiệm thu song song với giáo viên/kế toán; tài liệu tiếng Việt                                                                                                              |
| VPS thiếu tài nguyên khi chạy song song v2 + v3                   | Staging v3 nhỏ; nâng VPS trước giai đoạn D (chủ dự án đã sẵn sàng nâng dần)                                                                                                  |
