# Task 03 - Boundary Value Analysis: FR-10 - Order State Machine (Web)

Ngày lập: 2026-10-18 (Asia/Saigon). Repository: `E:/KiemThu/eshop-hw02`, origin `vinhuytran0810-cell/eshop-hw02`. Revision nền: `6ef7f1ee88a20ccc1b38c90e7ce12ba768f9df2d`. SRS: `README.md`, phiên bản 2.0. Đã đối chiếu source và cả bốn báo cáo `02_domain_testing.md`; báo cáo liên quan trực tiếp ở cùng thư mục với file này.

Phương pháp theo Task 03: 2-point chọn (L−1,L) và (U,U+1) ở biên nguyên, đúng một Valid tại biên và một Invalid ngay ngoài; 3-point chọn (B−1,B,B+1), phân Valid/Invalid theo SRS và loại trùng giá trị. Không áp công thức ±1 cho miền chưa có bước nhảy. Chỉ thay một Input Variable; các Input khác nominal, hợp lệ. Source chỉ xác định đường thao tác, không định nghĩa Expected Result. Các báo cáo Domain Testing được giữ nguyên. Chưa build/chạy ứng dụng, gửi API hoặc thực thi Test Case; không có Pass/Fail hoặc bug xác nhận.

SRS `README.md:143-162` (FR-10) định nghĩa năm trạng thái, các cạnh, final states và quyền hủy; không định nghĩa biến thời gian/số lượng trong State Machine. FR-12 dòng 176-179 bổ sung quyền Admin, không định nghĩa biên số. Source: `frontend-web/src/pages/Profile.jsx`, cancelOrder() dòng 66 và nút dòng 200; `frontend-admin/src/App.jsx`, updateOrderStatus() dòng 85, control dòng 814-869; `backend/server.js:321-342,525-568`, callback cancel/status; `backend/database.js:74-81`, orders. Không dùng INTEGER hoặc thứ tự câu if để suy ra miền nghiệp vụ có thứ tự.

## 1. Khả năng áp dụng BVA cho từng Input Variable

| Input | Khả năng áp dụng | Căn cứ / giải thích |
| --- | --- | --- |
| JWT / phiên | N/A | Không có tuổi token/độ dài được FR-10 đặc tả. |
| Actor / role | N/A | User/Admin là phân loại quyền. |
| Thao tác | N/A | Hủy/xác nhận/giao/hoàn tất là hành động, không phải khoảng số. |
| orderId / chủ đơn | N/A | Định danh và quyền sở hữu; SRS không quy định min/max ID. |
| currentStatus | N/A | pending, confirmed, shipping, delivered, canceled là nhãn. |
| requestedStatus | N/A | Nhãn trạng thái đích; chuyển hợp lệ phụ thuộc trạng thái nguồn và actor. |
| Quan hệ chuyển trạng thái | N/A | Miền đồ thị, không có cặp giá trị liền kề số học theo SRS. |

## 2. Boundary Identification Table

| Boundary ID | Biến | Loại biên | Giá trị B | Tính hợp lệ / bước liền kề | SRS |
| --- | --- | --- | --- | --- | --- |
| N/A | Tất cả Input FR-10 | N/A | Không có | Không có biến có thứ tự và ranh giới số từ SRS. | README.md:143-162 |

Đã hoàn thành mốc phân tích khả năng áp dụng và nhận diện biên. Phần thiết kế/coverage sẽ được bổ sung ở commit tiếp theo; chưa thực thi kiểm thử.
