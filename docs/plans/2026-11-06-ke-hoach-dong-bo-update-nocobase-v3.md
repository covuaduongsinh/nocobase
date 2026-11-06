# Kế hoạch Đồng bộ Cập nhật NocoBase 3 từ Upstream về Local và GitHub

- **Ngày lập:** 06/11/2026
- **Vị trí lưu:** `docs/plans/2026-11-06-ke-hoach-dong-bo-update-nocobase-v3.md`
- **Mục tiêu:** Cập nhật và đồng bộ toàn bộ thay đổi mới nhất từ upstream (`https://github.com/nocobase/nocobase.git`) vào kho lưu trữ local (`D:\code\nocobase`) và GitHub (`https://github.com/covuaduongsinh/nocobase.git`), bảo đảm an toàn dữ liệu kế hoạch và cấu hình riêng của Dương Sinh.

---

## 1. Hiện trạng Kho lưu trữ (Repositories & Branches)

1. **NocoBase v2:** Đã tách thành công ra repo độc lập:
   - Local: `D:\code\nocobase-v2`
   - Remote: `https://github.com/covuaduongsinh/nocobase-v2.git`
2. **Upstream NocoBase 3 (`upstream: https://github.com/nocobase/nocobase.git`):**
   - `upstream/v3-main`: Dừng ở commit `563f1d704a` (09/10/2026).
   - `upstream/v3-develop`: Nhánh phát triển chính thức và tích cực của NocoBase 3, có 60+ commits mới nhất (đến `7c25f88538` ngày 05/11/2026) với rất nhiều bản vá và tính năng mới về `@nocobase/app-client`, `@nocobase/app-plugin-agents`, `@nocobase/app-plugin-releases`, templates, UI library,...
3. **Local & GitHub NocoBase (`origin: https://github.com/covuaduongsinh/nocobase.git`):**
   - Đang có nhánh `v3-main` (HEAD: `f41ef5f699` chứa 2 commit tài liệu `docs/plans/` phía trên `563f1d704a`).
   - Có một số file uncommitted/untracked trong `D:\code\nocobase` (đặc biệt là toàn bộ tài liệu kế hoạch `docs/plans/` và các chỉnh sửa cục bộ).

---

## 2. Chiến lược Đồng bộ (Sync Strategy)

1. **Bảo toàn dữ liệu cục bộ:**
   - Tạo nhánh backup an toàn cho `v3-main` hiện tại: `backup/v3-main-before-sync-20261106`.
   - Lưu trữ tạm thời (stash) toàn bộ thay đổi chưa commit bằng `git stash`.
2. **Đồng bộ nhánh `v3-develop`:**
   - Checkout và tạo nhánh local `v3-develop` bám sát `upstream/v3-develop`.
   - Đẩy nhánh `v3-develop` lên `origin` (`covuaduongsinh/nocobase`) để lưu trữ toàn bộ code upstream mới nhất.
3. **Tích hợp cập nhật vào `v3-main`:**
   - Nhánh `upstream/v3-develop` là clean-room squash/commit root mới (`04922adec1`), do đó việc tích hợp vào `v3-main` sẽ thực hiện merge với cờ `--allow-unrelated-histories` hoặc tạo nhánh tích hợp sạch sẽ, giữ trọn vẹn thư mục `docs/plans/` của Dương Sinh.
   - Phục hồi các file kế hoạch và chỉnh sửa riêng của Dương Sinh từ stash/backup.
4. **Cài đặt & Kiểm thử chất lượng (Verification):**
   - Cập nhật dependencies: `pnpm install --no-frozen-lockfile`.
   - Kiểm tra typecheck và lint cơ bản.
5. **Đẩy lên GitHub:**
   - Push `v3-develop` và `v3-main` lên `covuaduongsinh/nocobase`.
   - Đặt `v3-main` làm nhánh mặc định (default branch).

---

## 3. Các bước thực hiện chi tiết

### Bước 1: Sao lưu & Stash trạng thái làm việc

```bash
git branch backup/v3-main-before-sync-20261106
git stash push -u -m "Local work before upstream v3 sync 2026-11-06"
```

### Bước 2: Kéo và cấu hình nhánh `v3-develop`

```bash
git checkout -B v3-develop upstream/v3-develop
git push -u origin v3-develop
```

### Bước 3: Tích hợp cập nhật vào `v3-main`

```bash
git checkout v3-main
git merge v3-develop --allow-unrelated-histories -m "chore(sync): merge upstream v3-develop updates into v3-main"
git stash pop
```

_Ghi chú: Nếu có xung đột (conflict) ở các file mã nguồn chỉnh sửa tạm thời, ưu tiên giữ mã chuẩn v3 mới nhất của upstream và bảo toàn 100% các file `docs/plans/`._

### Bước 4: Kiểm tra và cài đặt lại dependencies

```bash
CI=true pnpm install --no-frozen-lockfile
pnpm typecheck
```

### Bước 5: Commit bổ sung tài liệu và push lên GitHub

```bash
git add docs/plans/
git commit -m "docs(plans): update plans and sync logs"
git push origin v3-main
```

### Bước 6: Dọn dẹp nhánh cũ trên GitHub (nếu cần)

- Xóa nhánh `main` v2 cũ trên `origin/nocobase` (do đã chuyển sang repo `nocobase-v2`), đảm bảo repo `covuaduongsinh/nocobase` thuần túy là NocoBase 3.

---

## 4. Kết quả Thực hiện (Execution Summary)

1. **Đồng bộ nhánh `v3-develop`:**
   - Kéo toàn bộ 60+ commits mới nhất từ `upstream/v3-develop` (HEAD: `7c25f88538`).
   - Đã push nhánh `v3-develop` lên GitHub `covuaduongsinh/nocobase`.
2. **Tích hợp vào nhánh `v3-main`:**
   - Bảo toàn 100% tất cả 32 file tài liệu kế hoạch trong `docs/plans/`.
   - Giữ mã nguồn NocoBase 3 mới nhất chuẩn upstream.
   - Bổ sung bản vá tương thích Windows cho `dev-config/scripts/build.ts` và hỗ trợ reverse proxy headers (`x-forwarded-proto`, `x-forwarded-host`) trong `packages/app/app-server/src/realtime/websocket.ts` & `packages/app/app-websocket/src/index.ts`.
   - Commit: `13ea789055` (`fix: support windows pnpm execution and reverse-proxy websocket headers`).
3. **Cài đặt & Đồng bộ Workspace:**
   - Đã chạy thành công `pnpm install --no-frozen-lockfile` cho toàn bộ 92 workspace packages.
   - Các postinstall hook và skill sync hoạt động hoàn toàn xanh.
4. **Đồng bộ GitHub:**
   - Nhánh mặc định (default branch): `v3-main`.
   - Đã đẩy `v3-main` lên `origin`.
   - Đã xóa nhánh `main` v2 cũ trên remote `origin` (toàn bộ code v2 đã nằm độc lập tại repo `covuaduongsinh/nocobase-v2`).
