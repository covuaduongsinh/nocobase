# Prompt triển khai "Dương Sinh Studio" — dán vào phiên mới

> Dán toàn bộ khối dưới đường kẻ vào phiên Claude Code mới. Thư mục làm việc ban đầu: `D:\code\nocobase` (để đọc kế
> hoạch và tra cứu `upstream/v3-develop`); việc tạo ứng dụng sẽ diễn ra trong WSL. Không chứa mật khẩu / khoá API.

---

Bạn triển khai **Dương Sinh Studio** — bản tự dựng của NocoBase Studio (nền tảng điều phối coding agent) cho Công ty CP
Cờ vua Dương Sinh, trên **NocoBase 3**. Trả lời bằng tiếng Việt; code, lệnh, định danh giữ tiếng Anh; commit message
Conventional Commits tiếng Anh kèm dòng `Co-Authored-By`.

## Đọc trước (theo thứ tự, không bỏ qua)

1. `D:\code\nocobase\docs\plans\2026-11-01-nocobase-studio-dac-ta.md` — **Studio là gì**, phần nào mở/đóng, khái niệm,
   vòng đời issue, API, mô hình dữ liệu, quyền, cấu hình, CLI/giao thức, điểm mở rộng. Mục 0 là tóm tắt bắt buộc.
2. `D:\code\nocobase\docs\plans\2026-11-01-nocobase-studio-dac-ta-demo-api.md` — **giải mã demo quốc tế qua API**: bố cục
   màn hình, mô hình vai trò thật (owner/admin/contributor + catalog), cấu hình lead agent và nguyên tắc điều phối, chỉ dẫn
   agent thực thi, 11 skill, vòng đời issue thật, hợp đồng Git/PR/checks/preview/release, inbox/follow-ups/dashboard, CLI
   manifest. Kèm phụ lục `2026-11-01-nocobase-studio-phu-luc-api-cli.md` (653 route + 389 lệnh + schema) — dùng làm bản
   đối chiếu khi viết route `describeRoute()` và kiểm thử.
3. `D:\code\nocobase\docs\plans\2026-11-01-nocobase-studio-ke-hoach-phat-trien.md` — **kế hoạch** (quyết định kiến trúc
   A1–A10, mục 4b bản đồ 186 route Studio theo giai đoạn, lộ trình S0–S6, cấu trúc mã, kiểm chứng, rủi ro, 6 quyết định cần
   chốt, mục Tiến độ). Làm theo kế hoạch này.
4. Khi cần đối chiếu hành vi thật: demo https://studio.demo.nocobase.com/main/ (đăng nhập demo `POST /main/api/demoMode/accounts/signInDefault`,
   rồi `GET /main/api/swagger`, `GET /main/api/cli/manifest`, các route trong phụ lục). Chỉ đọc; không tạo dữ liệu trên demo.
5. `D:\code\nocobase\docs\plans\2026-10-30-ban-giao-phien-tiep-theo.md` — hiện trạng hệ thống NocoBase 2 của Dương Sinh,
   quyết định đã chốt, nơi để bí mật, bẫy Windows/NocoBase. Studio sẽ điều phối agent làm việc trên repo
   `D:\code\duongsinh-nocobase` này.
6. Mã nguồn tham chiếu NocoBase 3 (đã fetch nông, **chỉ đọc**, không checkout vào thư mục v2):
   `git -C D:\code\nocobase show upstream/v3-develop:<path>` hoặc `git -C D:\code\nocobase ls-tree -r --name-only upstream/v3-develop <dir>`
   (trong Git Bash đặt `MSYS_NO_PATHCONV=1` khi path có dấu `:`). Cần đọc:
   - `AGENTS.md`, `packages/templates/app-template-default/AGENTS.md`, `skills/nocobase-create-app/SKILL.md`
   - `packages/plugins/app-plugin-projects/{README.md,skills/nocobase-app-plugin-projects/SKILL.md,server/tokens.ts}`
   - `packages/plugins/app-plugin-agents/{README.md,skills/nocobase-app-plugin-agents/references/*.md,server/tokens.ts}`
   - `packages/plugins/app-plugin-releases/README.md`, `packages/plugins/app-plugin-knowledge/README.md`,
     `packages/plugins/app-plugin-agent-sessions/README.md`, `packages/app/agent-runner/README.md`,
     `packages/libs/agent-protocol/README.md`, `packages/libs/acp-bridge/README.md`, `packages/app/app-cli-client/README.md`
   - `.nb-studio/memory/*.md` (ví dụ định dạng project memory), `ui-library/registry/{projects,agents,inbox}/registry.json`
7. Tài liệu chính thức khi cần chi tiết hành vi: https://docs.nocobase.com/studio/… (bản zh-CN đầy đủ hơn bản EN ở các trang
   release/previews/versions).

## Cách làm việc

- **Ứng dụng mới ở repo riêng** `duongsinh-studio` (tạo trong WSL, `~/code/duongsinh-studio`, mirror git sang
  `D:\code\duongsinh-studio`; remote GitHub private). **Không** sửa `D:\code\nocobase` (chỉ tra cứu) và **không** đụng
  NocoBase 2 production.
- Tuân thủ `AGENTS.md` của ứng dụng NocoBase 3 vừa tạo (route trong `client/routes.ts`, migration tự chứa có `down`,
  mọi route tự `auth.required()`, chuỗi qua i18n `vi-VN` + `en-US`, shadcn/ui, `pnpm nocobase <topic> <cmd>`). Đọc
  `.agents/skills/nocobase-app-development/SKILL.md` của app trước khi viết code.
- Mỗi giai đoạn S0→S6: làm → `pnpm lint && pnpm typecheck && pnpm test && pnpm build` xanh → kiểm chứng theo mục 7 kế
  hoạch → commit → cập nhật mục **Tiến độ** trong file kế hoạch (`docs/plans/...ke-hoach-phat-trien.md` ở repo
  `D:\code\nocobase`) và `docs/` của repo studio. Dùng `superpowers:verification-before-completion` trước khi báo xong.
- Bí mật (GitHub App private key, SMTP, SECRETS_KEYS, token runner) chỉ nằm trong `config.yml`/biến môi trường/`*.secret`
  (gitignore); không in ra chat; không commit. Dùng `pnpm nocobase config set --from-env …`.
- Mọi hành động tốn tiền hoặc ra ngoài (tạo GitHub App/repo, đăng nhập coding tool trên VPS, cài runner trên VPS, gửi
  email thật): **hỏi chủ dự án trước**, dùng câu hỏi có lựa chọn và nêu khuyến nghị.
- Khi một gói `@nocobase/app-plugin-*@beta` không có trên npm hoặc API lệch với README đã đọc: dừng, so với
  `upstream/v3-develop` (có thể `git fetch --depth=1 upstream v3-develop` lại cho mới), ghi chênh lệch vào `docs/` của repo
  studio, rồi chọn cách ít rủi ro (ghim phiên bản hoặc build từ nguồn).
- Ưu tiên lắp từ plugin mở và UI library (`pnpm exec shadcn add @nocobase/<item>`); chỉ tự viết phần thuộc "lớp Studio"
  (mục 6 kế hoạch). Không mô phỏng lại các bảng plugin đã có.

## Bước đầu tiên

1. Hỏi chủ dự án **6 quyết định ở mục 9** của kế hoạch (tự dựng vs mua; Git hosting; vị trí runner shared; tài khoản
   coding tool; SMTP; thứ tự giai đoạn). Khuyến nghị kèm theo đã ghi trong kế hoạch.
2. Thực hiện **S0** theo checklist 5.S0: kiểm tra WSL Ubuntu, Node 24, pnpm 11 (máy Windows hiện có Node 25.2.1, **chưa có
   pnpm**, chưa có Docker); tạo app bằng `pnpm create @nocobase/app@beta duongsinh-studio --template-tag=beta --json`;
   `config init --dialect sqlite`; `pnpm dev` ở cổng 13100; đăng ký plugin projects/agents/agent-sessions/knowledge/
   releases; commit đầu tiên; cập nhật Tiến độ.
3. Sang **S1** (lõi dự án & issue) rồi S2, S3 — mục tiêu MVP là **một issue thật trên `duongsinh-nocobase` đi trọn 8 bước
   (yêu cầu → lead agent giao → agent dev trên runner → PR → checks → review chéo → chủ dự án Merge trong Studio → Done →
   project memory)**.

## Điều kiện dừng/báo cáo

- Cuối mỗi giai đoạn: báo ngắn gọn bằng tiếng Việt — đã làm gì, bằng chứng kiểm chứng (lệnh + kết quả), gì còn mở, câu
  hỏi cho chủ dự án. Không báo "đã xong" khi còn `TODO`, test bị skip, hoặc chưa chạy `pnpm build`.
- Nếu bị chặn bởi thượng nguồn (gói chưa publish, lỗi beta), ghi rõ commit `v3-develop` đã đối chiếu và phương án vòng.
