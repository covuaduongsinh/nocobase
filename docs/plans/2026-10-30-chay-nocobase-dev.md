# Kế hoạch: tìm hiểu, giới thiệu và chạy NocoBase (dev từ source, SQLite)

Bản chính thức sẽ được lưu tại `docs/plans/2026-10-30-chay-nocobase-dev.md` sau khi duyệt (quy ước mới: mọi kế hoạch để ở `docs/plans/`).

## Bối cảnh

Repo `D:\code\nocobase` là NocoBase 2.2.36 (Apache-2.0), nền tảng AI + no-code mã nguồn mở. Máy Windows 10, chưa có `node_modules`, `.env`, bản build; `storage/` rỗng. Mục tiêu: hiểu phần mềm và chạy được bản dev với SQLite.

## Giới thiệu chi tiết

- Kiến trúc microkernel: lõi `packages/core` (27 gói: server, database, acl, auth, resourcer, cache, logger..., client, client-v2, flow-engine, cli, build), gần như mọi tính năng là plugin.
- `packages/plugins/@nocobase`: khoảng 110 plugin (workflow và ~25 node, block/field/action, auth/users/departments, file-manager, data-source, multi-app, backup-restore, ai, mcp-server, mobile...).
- `packages/presets/nocobase`: bộ plugin mặc định.
- Hai client runtime: v1 legacy (`SchemaComponent`, phục vụ ở gốc) và v2 mới (`FlowEngine`/`FlowModel`, tiền tố `/v`). v1 được import v2, không ngược lại.
- Sẽ viết thêm 1 mục giới thiệu ngắn bằng tiếng Việt cho người dùng sau khi chạy xong (quản trị, collection, trang, workflow, plugin manager).

## Các bước chạy (chỉ thực hiện sau khi duyệt)

1. Kiểm tra: `node -v` (cần >= 22), `yarn -v` (1.22.x), git. Thiếu thì cài Node 22 và `npm i -g yarn`.
2. Windows: bật Developer Mode (symlink của yarn workspaces) và `LongPathsEnabled`; đặt repo ở đường dẫn ngắn nếu lỗi.
3. `yarn install` (lâu, có `postinstall`, `patch-package`; lưu ý lỗi native từ `plugin-ai` shim).
4. Tạo `.env` từ `.env.example`: `DB_DIALECT=sqlite`, `DB_STORAGE=storage/db/nocobase.sqlite`, giữ `APP_PORT=13000`, đổi `APP_KEY` thành chuỗi ngẫu nhiên; bỏ/ghi chú các biến DB_HOST/PORT của postgres.
5. `yarn nocobase install` (khởi tạo DB và tài khoản root: `admin@nocobase.com` / `admin123`, nên đổi sau).
6. `yarn dev` rồi mở `http://localhost:13000` (v2: `http://localhost:13000/v`). Nếu `yarn dev` (rsbuild) lỗi, thử `yarn dev:umi`.
7. Đăng nhập, xem Plugin manager, tạo thử một collection và một trang.

## Rủi ro đã biết

- SQLite có trong code (`packages/core/database/src/dialects/sqlite-dialect.ts`) nhưng chưa xác nhận là hỗ trợ chính thức cho dev; đối chiếu `docs/docs/en/api/cli/init.md` khi thực hiện. Nếu lỗi, chuyển sang Postgres qua `docker-compose.yml`.
- `volta` trong `package.json` ghi Node 20 nhưng `engines` yêu cầu >= 22: theo `engines`.
- Lỗi build native trên Windows, symlink, đường dẫn dài.

## Kiểm chứng

- `yarn install` thoát mã 0; `yarn nocobase install` báo thành công.
- `yarn dev` lên cổng 13000, trang đăng nhập tải được, đăng nhập thành công.
- API trả lời (ví dụ `GET /api/__health_check`).
- Không sửa file mã nguồn; chỉ tạo `.env`, `node_modules`, `storage/db`. Không commit.
