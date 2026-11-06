# Prompt triển khai giai đoạn 2 — dán vào phiên mới

> Dán toàn bộ khối dưới đường kẻ vào phiên Claude Code mới (thư mục làm việc `D:\code\nocobase`). Không chứa mật khẩu / khoá API.

---

Bạn triển khai **giai đoạn 2** của hệ thống quản lý Công ty CP Cờ vua Dương Sinh trên NocoBase: **cổng phụ huynh, Zalo OA/ZNS, thanh toán VietQR tự ghi nhận, tuyển sinh**. Trả lời bằng tiếng Việt.

## Đọc trước (theo thứ tự, không bỏ qua)

1. `D:\code\nocobase\docs\plans\2026-10-30-ban-giao-phien-tiep-theo.md` — hiện trạng, quyết định đã chốt, nơi để bí mật, cách chạy, **bẫy môi trường Windows và bẫy NocoBase** đã gặp.
2. `D:\code\nocobase\docs\plans\2026-10-31-giai-doan-2-cong-phu-huynh-zalo-vietqr.md` — **kế hoạch giai đoạn 2** (thiết kế, lộ trình 2.0–2.6, rủi ro, kiểm chứng). Làm theo kế hoạch này.
3. `D:\code\nocobase\docs\plans\2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md` — mục "Tiến độ".
4. `D:\code\duongsinh-nocobase\README.md`, rồi `git -C D:\code\duongsinh-nocobase status` và `log --oneline | head -20`.

## Cách làm việc

- Mọi thay đổi bằng script tái chạy an toàn trong `D:\code\duongsinh-nocobase\setup\` (schema `setup/schema/phase1.mjs`, vai trò `setup/acl/apply.mjs`, workflow `setup/workflows/apply.mjs`, giao diện `setup/ui/*.mjs`, kiểm thử `setup/verify.mjs`). Không sửa lõi NocoBase trong `D:\code\nocobase`.
- Mỗi bước: thử ở dev local (SQLite, Node 22 — xem file bàn giao) → production `https://nocobase.dsc.edu.vn` → `node setup/verify.mjs` phải "tất cả đạt" → sao lưu (`ssh root@217.15.160.118 /usr/local/bin/nocobase-dsc-backup.sh`) → commit Conventional Commits (tiếng Anh, có dòng `Co-Authored-By`) → cập nhật mục Tiến độ + file bàn giao.
- Sao lưu **trước** mọi thay đổi dữ liệu hàng loạt trên production. Không in mật khẩu / token ra chat; bí mật để trong file `*.secret` (đã gitignore) hoặc biến môi trường NocoBase.
- Không dùng dữ liệu tài chính thật để thử; webhook thử bằng payload mẫu. ZNS bật ở **chế độ thử** (chỉ gửi đến số của chủ dự án) cho tới khi chủ dự án duyệt.
- Mọi việc gửi tin nhắn thật cho phụ huynh, cấp tài khoản thật hàng loạt, hay đăng ký dịch vụ trả phí: **hỏi chủ dự án trước**.

## Bước đầu tiên

Hỏi chủ dự án 5 quyết định ở mục 6 của kế hoạch giai đoạn 2 (dùng câu hỏi có lựa chọn, ghi rõ khuyến nghị):

1. Đăng nhập phụ huynh: mật khẩu tạm trước (khuyên) hay chờ OTP.
2. Tài khoản ngân hàng nhận học phí + cổng tự ghi nhận (SePay / Casso).
3. Zalo OA đã xác thực chưa, ai duyệt mẫu ZNS.
4. Phụ huynh được xem những gì (nhận xét, phiếu thu chi tiết).
5. Thứ tự ưu tiên các bước (khuyên 2.1 → 2.2 → 2.3 → 2.4 → 2.5).

Sau đó bắt đầu **bước 2.1 — Cổng phụ huynh** (không phụ thuộc tài khoản bên ngoài), với yêu cầu tối thượng: **kiểm thử tự động cách ly dữ liệu giữa 2 gia đình trên mọi collection trước khi tạo bất kỳ tài khoản phụ huynh thật nào.**
