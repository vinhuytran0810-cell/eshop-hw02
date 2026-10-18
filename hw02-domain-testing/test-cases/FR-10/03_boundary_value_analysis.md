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

## 3. Bảng 2-point BVA và 3-point BVA

### 2-point BVA

| Biên | Bộ giá trị | Valid/Invalid | Kết luận |
| --- | --- | --- | --- |
| N/A | Không có | N/A | Không áp dụng; không tạo giá trị/Test Case giả. |

### 3-point BVA

| Biên | Bộ giá trị | Valid/Invalid | Kết luận |
| --- | --- | --- | --- |
| N/A | Không có | N/A | Không áp dụng; không tạo giá trị/Test Case giả. |

Không gán pending=1, confirmed=2,... để dùng ±1. Trạng thái shipping/final state có ý nghĩa ranh giới hành vi nhưng không phải BVA số học. Không nhập các Test Cases State Transition đã có trong 02_domain_testing.md rồi đổi nhãn thành BVA. Các thông tin order amount, quantity hoặc login lockout thuộc chức năng khác, không kéo vào chỉ để có Boundary Values cho FR-10.

## 4. BVA Test Cases

**N/A - không có BVA Test Cases.** Không gán ID giả, không chạy ca Domain Testing/State Transition dưới tên BVA; 0 ca là kết quả phân tích có căn cứ, không phải thiếu thiết kế.

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| N/A | N/A | N/A | Không thực thi | BVA không áp dụng theo SRS hiện tại | N/A |

## 5. Boundary Coverage Matrix

| Input / phạm vi | 2-point | 3-point | Test Cases | Kết luận |
| --- | --- | --- | --- | --- |
| JWT / phiên | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| Actor / role | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| Thao tác | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| orderId / chủ đơn | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| currentStatus | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| requestedStatus | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| Quan hệ chuyển trạng thái | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |

Coverage BVA là N/A, không ghi 100% hoặc giả định có boundary được kiểm thử.

## 6. Giả định, giới hạn và các điểm cần human review

1. Kết luận BVA N/A áp dụng với SRS hiện tại, không có nghĩa FR-10 không cần kiểm thử: Domain Testing và State Transition Coverage Matrix vẫn là tài liệu phù hợp.
2. Admin shipping→canceled và self-loop còn cần human review như báo cáo Domain Testing; giải quyết chúng không tạo ra bước nhảy số học.
3. Không giả định timeout giao hàng, giới hạn số lần đổi status hoặc min/max ID. Source không phải nguồn bổ sung các ràng buộc này.
4. Phân biệt User Web, Web Admin và Backend API như báo cáo Domain Testing; không chạy các tầng này trong nhiệm vụ BVA.

Đã hoàn thành thiết kế Task 03 cho chức năng này. Dừng để human review; chưa chuyển sang Test Execution.
