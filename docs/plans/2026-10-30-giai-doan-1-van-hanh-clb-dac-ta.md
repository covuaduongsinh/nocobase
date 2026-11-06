# Đặc tả giai đoạn 1 — Vận hành CLB Dương Sinh trên NocoBase

Thuộc kế hoạch tổng: `docs/plans/2026-10-30-chien-luoc-he-thong-duong-sinh-nocobase.md`. Ngày lập: 30/10/2026.

## 1. Phạm vi

Thay bảng tính/Lark cho các việc hằng ngày của CLB: hồ sơ học viên – phụ huynh, lớp và buổi học, **điểm danh kèm nhận xét**, gói học phí và số buổi còn lại, phiếu thu, thẻ sao, lead học thử, phân quyền theo vai trò. Không thuộc giai đoạn này: cổng phụ huynh, Zalo OA, VietQR tự động, LMS.

Toàn bộ cấu trúc được dựng bằng script tái chạy được (idempotent) ở repo riêng `D:\code\duongsinh-nocobase` (thư mục `setup/`), gọi API `collections:apply` của NocoBase. Collection tạo ra vẫn hiển thị và sửa được trong trình quản lý dữ liệu trên giao diện.

## 2. Quy ước

- Tên collection có tiền tố `dsc_`, tên trường `snake_case` không dấu; tiêu đề hiển thị tiếng Việt có dấu.
- Mọi collection dùng template `general` (có `id`, `createdAt/By`, `updatedAt/By`, bật nhật ký).
- Quan hệ đều tạo trường ngược (ví dụ `dsc_hoc_vien.diem_danh`) để xem chi tiết hai chiều.

## 3. Mô hình dữ liệu

| Collection          | Tiêu đề              | Trường chính                                                                                                                                                                                         |
| ------------------- | -------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `dsc_co_so`         | Cơ sở                | `ten`, `ma`, `loai` (trụ sở/CLB/trường đối tác/nhượng quyền), `dia_chi`, `dien_thoai`, `dang_hoat_dong`                                                                                              |
| `dsc_cap_do`        | Cấp độ               | `ma` (tot…vua), `ten`, `thu_tu`, `dinh_huong`, `chuan_dau_ra`                                                                                                                                        |
| `dsc_nhan_su`       | Nhân sự              | `ho_ten`, `vai_tro`, `dien_thoai`, `email`, `trang_thai`, → `co_so`, → `tai_khoan` (users)                                                                                                           |
| `dsc_phu_huynh`     | Phụ huynh            | `ho_ten`, `dien_thoai`, `zalo`, `email`, `quan_he`, `dia_chi`, `dong_y_du_lieu`, `ngay_dong_y`, `ghi_chu`, ↔ `hoc_vien`                                                                              |
| `dsc_hoc_vien`      | Học viên             | `ma_hoc_vien` (tự sinh HV00001), `ho_ten`, `ngay_sinh`, `gioi_tinh`, `truong`, `lop_o_truong`, `trang_thai`, `ngay_nhap_hoc`, `tong_sao`, `ghi_chu`, → `co_so`, → `cap_do`                           |
| `dsc_lop_hoc`       | Lớp học              | `ten`, `lich_hoc`, `phong`, `si_so_toi_da`, `trang_thai`, → `co_so`, → `cap_do`, → `giao_vien`, → `tro_giang`, ↔ `hoc_vien`                                                                          |
| `dsc_buoi_hoc`      | Buổi học             | `ngay`, `gio_bat_dau`, `bai_hoc`, `noi_dung`, `trang_thai`, → `lop`, → `giao_vien`                                                                                                                   |
| `dsc_diem_danh`     | Điểm danh & nhận xét | `trang_thai` (có mặt/vắng có phép/vắng không phép/học bù), `tu_duy`, `tap_trung`, `tinh_than` (thang 1–5), `nhan_xet`, `sao_thuong`, → `buoi_hoc`, → `hoc_vien`, → `dang_ky_goi`                     |
| `dsc_goi_hoc`       | Gói học phí          | `ten`, `so_buoi`, `hoc_phi`, `thoi_han_ngay`, `qua_tang`, `dang_ap_dung`                                                                                                                             |
| `dsc_dang_ky_goi`   | Đăng ký gói          | `ma` (DK00001), `ngay_bat_dau`, `han_su_dung`, `so_buoi`, `so_buoi_da_hoc`_, `so_buoi_con_lai`_, `hoc_phi`, `giam_gia`, `da_thu`*, `trang_thai`, `da_tang_qua`, → `hoc_vien`, → `goi_hoc`, → `co_so` |
| `dsc_phieu_thu`     | Phiếu thu            | `so_phieu` (PT00001), `ngay_thu`, `so_tien`, `hinh_thuc`, `trang_thai`, `ghi_chu`, → `dang_ky_goi`, → `hoc_vien`, → `nguoi_thu`                                                                      |
| `dsc_giao_dich_sao` | Giao dịch thẻ sao    | `so_sao` (+/−), `loai`, `ly_do`, → `hoc_vien`, → `diem_danh`                                                                                                                                         |
| `dsc_lead`          | Lead tuyển sinh      | `ten_phu_huynh`, `dien_thoai`, `ten_hoc_vien`, `nam_sinh`, `nguon`, `trang_thai`, `lich_hoc_thu`, `ghi_chu`, → `co_so`, → `nguoi_phu_trach`, → `hoc_vien`                                            |

`*` = do workflow tự tính, không nhập tay.

**Gộp điểm danh và nhận xét vào một bản ghi**: mỗi học viên mỗi buổi chỉ có một dòng, giáo viên chọn trạng thái, chấm 3 tiêu chí, ghi một câu nhận xét và số sao thưởng. Mục tiêu dưới 2 phút cho mỗi học viên.

## 4. Dữ liệu khởi tạo

- 3 cơ sở: Trụ sở Tôn Đức Thắng, CLB Kim Liên, CLB Vĩnh Phúc.
- 6 cấp độ Tốt → Vua kèm định hướng theo hồ sơ năng lực.
- 3 gói: 10 buổi 1.500.000đ, 20 buổi 3.000.000đ (tặng bộ cờ), 40 buổi 5.500.000đ (tặng domino + miễn giảm sách).
- Nhân sự: chỉ tạo khi chủ dự án xác nhận (dữ liệu cá nhân). Script không tự tạo người.

## 5. Quy tắc nghiệp vụ (workflow)

| #   | Tên                      | Kích hoạt                                    | Xử lý                                                                                                                                                       |
| --- | ------------------------ | -------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| W1  | Tính lại số buổi của gói | Tạo/sửa `dsc_diem_danh`                      | Đếm điểm danh có tính buổi của gói → cập nhật `so_buoi_da_hoc`, `so_buoi_con_lai`; ≤ 2 buổi → trạng thái "Sắp hết" và thông báo trong app; ≤ 0 → "Hết buổi" |
| W2  | Cộng sao từ điểm danh    | Tạo `dsc_diem_danh` có `sao_thuong` > 0      | Tạo giao dịch sao loại "Thưởng buổi học"                                                                                                                    |
| W3  | Tính tổng sao            | Tạo/sửa `dsc_giao_dich_sao`                  | Tổng `so_sao` của học viên → `dsc_hoc_vien.tong_sao`                                                                                                        |
| W4  | Tính đã thu              | Tạo/sửa `dsc_phieu_thu`                      | Tổng phiếu "Đã thu" của gói → `dsc_dang_ky_goi.da_thu`                                                                                                      |
| W5  | Đóng gói hết hạn         | Hằng ngày 06:00                              | Gói đang học có `han_su_dung` < hôm nay → "Hết hạn"                                                                                                         |
| W6  | Tạo sẵn điểm danh        | Buổi học chuyển "Đã dạy" (nút ▶ trên mobile) | Nếu buổi chưa có điểm danh: mỗi học viên của lớp một dòng "Có mặt", gắn gói đang học cũ nhất                                                                |

**Quy tắc trừ buổi (chủ dự án chốt 30/10/2026):** chỉ trạng thái _có mặt_ mới trừ buổi; _học bù_, _vắng có phép_, _vắng không phép_ đều không trừ. Buổi học bù mà muốn trừ vào gói thì điểm danh là _có mặt_. Gói có thời hạn tính bằng ngày (`thoi_han_ngay`, mặc định để trống = không giới hạn).

## 6. Phân quyền

| Vai trò (`name`)                        | Quyền                                                                       |
| --------------------------------------- | --------------------------------------------------------------------------- |
| `dsc_quan_ly` (Quản lý)                 | Toàn quyền dữ liệu `dsc_*`                                                  |
| `dsc_giao_vien` (Giáo viên / trợ giảng) | Xem học viên, lớp, gói; thêm/sửa buổi học và điểm danh; không xem phiếu thu |
| `dsc_ke_toan` (Kế toán)                 | Xem tất cả; thêm/sửa gói, đăng ký gói, phiếu thu                            |
| `dsc_tu_van` (Tư vấn tuyển sinh)        | Toàn quyền lead; xem/thêm học viên, phụ huynh                               |

Giới hạn "giáo viên chỉ thấy lớp mình" (data scope theo `{{$user.id}}`) sẽ cấu hình khi đã liên kết tài khoản nhân sự.

## 7. Giao diện (cấu hình trên UI sau khi có dữ liệu)

1. Menu **Học viên**: bảng học viên + bộ lọc cơ sở/cấp/trạng thái; trang chi tiết có tab phụ huynh, gói học, điểm danh, sao.
2. Menu **Lớp & buổi học**: lịch (calendar) buổi học; trong buổi học có bảng điểm danh sửa trực tiếp.
3. Menu **Học phí**: đăng ký gói (lọc "Sắp hết"/"Hết hạn"), phiếu thu có nút in.
4. Menu **Tuyển sinh**: kanban lead theo trạng thái; form công khai đăng ký học thử.
5. **Dashboard**: học viên đang học theo cơ sở/cấp, chuyên cần tháng, doanh thu tháng, gói sắp hết.

## 8. Kiểm chứng

- Script chạy 2 lần liên tiếp không lỗi, không tạo trùng.
- Tạo 1 học viên + gói 10 buổi + 3 điểm danh (2 có mặt, 1 vắng có phép) → `so_buoi_da_hoc = 2`, `so_buoi_con_lai = 8`.
- Điểm danh có `sao_thuong = 2` → có giao dịch sao, `tong_sao` tăng 2.
- Phiếu thu 1.500.000đ "Đã thu" → `da_thu = 1500000`.
- Đăng nhập bằng tài khoản vai trò giáo viên: không đọc được `dsc_phieu_thu`.
