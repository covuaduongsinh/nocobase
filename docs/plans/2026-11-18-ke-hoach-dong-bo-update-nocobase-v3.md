# Kế hoạch Đồng bộ Cập nhật NocoBase 3 từ Upstream về Local và GitHub

- **Ngày lập:** 18/11/2026
- **Vị trí lưu:** `docs/plans/2026-11-18-ke-hoach-dong-bo-update-nocobase-v3.md`
- **Mục tiêu:** Cập nhật và đồng bộ toàn bộ thay đổi mới nhất từ upstream (`https://github.com/nocobase/nocobase.git`) vào kho lưu trữ local (`D:\code\nocobase`) và GitHub (`https://github.com/covuaduongsinh/nocobase.git`), bảo đảm an toàn dữ liệu kế hoạch tại `docs/plans/` và các bản vá tương thích Windows của Dương Sinh.

---

## 1. Hiện trạng Kho lưu trữ (Repositories & Branches)

1. **Remote cấu hình:**
   - `origin`: `https://github.com/covuaduongsinh/nocobase.git`
   - `upstream`: `https://github.com/nocobase/nocobase.git`

2. **Upstream NocoBase 3:**
   - Đợt sync trước (06/11/2026): dừng ở commit `7c25f88538` trên nhánh `v3-develop`.
   - Thay đổi lớn từ Upstream: Nhánh phát triển chính thức đã được đổi tên từ `v3-develop` sang `v3` (commit `a2474edde5`).
   - Nhánh `upstream/v3`: Hiện tại có **154 commits mới** (từ `7c25f88538` đến `e90a91e860`, bao gồm các bản phát hành `2026-11-16.1`, `2026-11-17.1`, `2026-11-17.2` với tag `nocobase-v3-beta/2026-11-17.2`).
   - Các tính năng & bản vá chính từ upstream:
     - Knowledge plugin: render markdown với MarkdownView dùng chung, mở link tab mới (`#11598`).
     - Database core: xử lý NULL temporal values (`#11597`), cảnh báo thiếu migration source & preflight rollback (`#11595`).
     - Agent / AI: runner retry authorization (`#11584`), runtime idle conversations (`#11577`), stream diagnostics & export logs (`#11563`), initiator grants (`#11554`).
     - App Host: phát hiện listener ports qua IPC (`#11576`), cấu hình worker deadlines (`#11578`).
     - UI Library: workflow version selector (`#11569`), template permissions & public forms.
     - Studio CE: chuẩn hoá Docker release builds & license (`#11567`, `#11593`).

3. **Local & GitHub NocoBase (`covuaduongsinh/nocobase`):**
   - Nhánh `v3-main`: Đang ở `d4f6b5a78c` (chứa các commit tài liệu `docs/plans/` và bản vá Windows/reverse-proxy tại commit `13ea789055`).
   - Các file có thay đổi chưa commit cục bộ:
     - `packages/app/app-cli/src/tools/scripts/utils/retarget-native.mjs`: sửa lỗi thực thi lệnh npm/pnpm trên Windows (`spawnSync` với `.cmd` và `shell: true`).
     - `docs/plans/2026-11-01-nocobase-studio-ke-hoach-phat-trien.md`: bổ sung ghi nhận đợt cập nhật U1–U8.

---

## 2. Chiến lược Đồng bộ (Sync Strategy)

1. **Bảo toàn dữ liệu cục bộ & Tạo điểm khôi phục:**
   - Commit các thay đổi cục bộ hợp lệ (`retarget-native.mjs` và cập nhật file kế hoạch Studio).
   - Tạo nhánh sao lưu: `backup/v3-main-before-sync-20261118`.
2. **Đồng bộ nhánh nguồn `v3` từ Upstream:**
   - Tạo / cập nhật nhánh local `v3` bám sát `upstream/v3` (`e90a91e860`).
   - Đẩy nhánh `v3` lên remote `origin` (`covuaduongsinh/nocobase`).
3. **Tích hợp cập nhật Upstream vào `v3-main`:**
   - Merge `v3` vào `v3-main`. Vì `upstream/v3` là nhánh kế thừa trực tiếp từ commit `7c25f88538` mà `v3-main` đang đứng trên đó, và upstream không đụng tới các file bản vá Windows hay `docs/plans/`, việc merge sẽ diễn ra tự động và sạch sẽ.
4. **Cài đặt & Kiểm thử (Verification):**
   - Chạy `pnpm install --no-frozen-lockfile` để cập nhật dependencies và liên kết workspace packages.
   - Kiểm tra build / typecheck các package cốt lõi.
5. **Đồng bộ lên GitHub:**
   - Đẩy `v3` và `v3-main` lên `origin`.
   - Giữ `v3-main` là nhánh làm việc chính.

---

## 3. Các bước triển khai chi tiết

### Bước 1: Commit thay đổi cục bộ và sao lưu trạng thái

```bash
git add packages/app/app-cli/src/tools/scripts/utils/retarget-native.mjs docs/plans/2026-11-01-nocobase-studio-ke-hoach-phat-trien.md docs/plans/2026-11-18-ke-hoach-dong-bo-update-nocobase-v3.md
git commit -m "feat(app-cli): support windows pnpm in retarget-native and update plans"
git branch backup/v3-main-before-sync-20261118
```

### Bước 2: Kéo và cấu hình nhánh `v3` từ Upstream

```bash
git checkout -B v3 upstream/v3
git push -u origin v3
```

### Bước 3: Tích hợp cập nhật vào `v3-main`

```bash
git checkout v3-main
git merge v3 -m "chore(sync): merge upstream v3 updates (154 commits to 2026-11-17.2) into v3-main"
```

### Bước 4: Cài đặt dependencies và kiểm tra chất lượng

```bash
CI=true pnpm install --no-frozen-lockfile
```

### Bước 5: Đẩy thay đổi lên GitHub

```bash
git push origin v3-main
git push origin v3 --tags
```

---

## 4. Kết quả Thực hiện (Execution Summary)

_(Sẽ cập nhật sau khi hoàn thành các bước triển khai)_
