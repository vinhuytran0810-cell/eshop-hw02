# Task 03 - Boundary Value Analysis: FR-15 - Product CRUD (Web Admin)

Ngày lập: 2026-10-18 (Asia/Saigon). Repository: `E:/KiemThu/eshop-hw02`, origin `vinhuytran0810-cell/eshop-hw02`. Revision nền: `6ef7f1ee88a20ccc1b38c90e7ce12ba768f9df2d`. SRS: `README.md`, phiên bản 2.0. Đã đối chiếu source và cả bốn báo cáo `02_domain_testing.md`; báo cáo liên quan trực tiếp ở cùng thư mục với file này.

Phương pháp theo Task 03: 2-point chọn (L−1,L) và (U,U+1) ở biên nguyên, đúng một Valid tại biên và một Invalid ngay ngoài; 3-point chọn (B−1,B,B+1), phân Valid/Invalid theo SRS và loại trùng giá trị. Không áp công thức ±1 cho miền chưa có bước nhảy. Chỉ thay một Input Variable; các Input khác nominal, hợp lệ. Source chỉ xác định đường thao tác, không định nghĩa Expected Result. Các báo cáo Domain Testing được giữ nguyên. Chưa build/chạy ứng dụng, gửi API hoặc thực thi Test Case; không có Pass/Fail hoặc bug xác nhận.

SRS `README.md:193-198`: Admin CRUD; name bắt buộc, tối đa 255 ký tự (P15-02); price bắt buộc và >0 (P15-03); category chọn từ danh sách hiện có (P15-04); Update chỉ thay sản phẩm đích (P15-05). FR-12 dòng 176-179 cho JWT/role mutations. Source: `frontend-admin/src/App.jsx`, handleProductSubmit() dòng 105-131, form 483-543, bảng 567-608; `backend/server.js:141-196,243-247`, callbacks products/categories; schema `backend/database.js:64-71`. HTML type=number, kiểu INTEGER và validation hiện tại không tạo quy tắc integer-only/step giá mới.

## 1. Khả năng áp dụng BVA cho từng Input Variable

| Input | Khả năng áp dụng | Căn cứ / giải thích |
| --- | --- | --- |
| JWT | N/A | Credential phân loại; không có độ dài/thời hạn đặc tả. |
| role | N/A | Quyền phân loại; giữ Admin hợp lệ. |
| Operation | N/A cho bản thân operation | Create/Update là hai ngữ cảnh kiểm tra cùng biên name; Read/Delete không có input tên mới. |
| productId | N/A | Định danh có membership; không có min/max nghiệp vụ. |
| name | Áp dụng theo độ dài L | Required diễn giải là chuỗi không rỗng có nội dung; chọn ASCII A lặp L lần, 1≤L≤255. |
| price | Nhận diện được điểm cắt; chưa đủ bộ BVA chuẩn | 0 Invalid, >0 Valid nhưng không có precision/step; không xác định duy nhất điểm liền kề. |
| category_id | N/A | Membership trong danh sách có sẵn, không phải khoảng ID. |
| description | N/A | Không có min/max trong FR-15. |
| imageUrl | N/A | Không có giới hạn chuỗi/URL từ FR-15. |
| Tập sản phẩm đối chứng | N/A | Preconditions để quan sát tính độc lập, không phải miền số để kiểm biên. |

## 2. Boundary Identification Table

| Boundary ID | Biến | Loại biên | Giá trị B | Tính hợp lệ / bước liền kề | SRS |
| --- | --- | --- | --- | --- | --- |
| FR15-B-L | L = độ dài name | Dưới | 1 | Inclusive, Valid tại biên; bước độ dài một ký tự ASCII. | P15-02, README.md:195 |
| FR15-B-U | L = độ dài name | Trên | 255 | Inclusive, Valid tại biên; bước độ dài một ký tự ASCII. | P15-02, README.md:195 |
| FR15-B-P0 | price | Điểm cắt bị loại | 0 | 0 Invalid; >0 Valid; chưa xác định step/giá liền kề. Bộ BVA chuẩn chưa đủ cơ sở. | P15-03, README.md:196 |

Đã hoàn thành mốc phân tích khả năng áp dụng và nhận diện biên. Phần thiết kế/coverage sẽ được bổ sung ở commit tiếp theo; chưa thực thi kiểm thử.
