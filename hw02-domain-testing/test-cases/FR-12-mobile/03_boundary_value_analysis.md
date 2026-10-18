# Task 03 - Boundary Value Analysis: FR-12 - Access Control (Mobile)

Ngày lập: 2026-10-18 (Asia/Saigon). Repository: `E:/KiemThu/eshop-hw02`, origin `vinhuytran0810-cell/eshop-hw02`. Revision nền: `6ef7f1ee88a20ccc1b38c90e7ce12ba768f9df2d`. SRS: `README.md`, phiên bản 2.0. Đã đối chiếu source và cả bốn báo cáo `02_domain_testing.md`; báo cáo liên quan trực tiếp ở cùng thư mục với file này.

Phương pháp theo Task 03: 2-point chọn (L−1,L) và (U,U+1) ở biên nguyên, đúng một Valid tại biên và một Invalid ngay ngoài; 3-point chọn (B−1,B,B+1), phân Valid/Invalid theo SRS và loại trùng giá trị. Không áp công thức ±1 cho miền chưa có bước nhảy. Chỉ thay một Input Variable; các Input khác nominal, hợp lệ. Source chỉ xác định đường thao tác, không định nghĩa Expected Result. Các báo cáo Domain Testing được giữ nguyên. Chưa build/chạy ứng dụng, gửi API hoặc thực thi Test Case; không có Pass/Fail hoặc bug xác nhận.

SRS FR-12, `README.md:176-179`: Admin subsystem chỉ role=admin; tất cả /api/admin/* và product/category/coupon mutations yêu cầu JWT hợp lệ AND role=admin. Không tạo FR-12 riêng cho Mobile. Source: `frontend-mobile/App.js:16,34-35,161-207,759,893,978-985`, API_URL, token/user state, logout(), handleLogin(), renderLogin(), renderProfile() và chuyển view; không có Admin UI/control hoặc lời gọi protected Admin CRUD. Backend: `backend/server.js:51,100-110,167-196,199,249-277,457-568`, JWT issuance, authenticateToken() và route callbacks; không có thời hạn token rõ trong lời gọi jwt.sign. Quan sát implementation không bổ sung chính sách lifetime/length vào SRS.

## 1. Khả năng áp dụng BVA cho từng Input Variable

| Input | Khả năng áp dụng | Căn cứ / giải thích |
| --- | --- | --- |
| JWT / Authorization | N/A | Hợp lệ/thiếu/sai là phân loại; SRS không có giới hạn chiều dài hay lifetime. |
| role claim | N/A | admin hay không phải admin là phân loại quyền; không tự đánh số. |
| Protected method/route | N/A | Membership trong 13 routes đã xác minh ở Domain Testing, không phải khoảng số endpoint. |
| HTTP method | N/A | GET/POST/PUT/DELETE là nhãn; không có giá trị liền kề số học. |
| Body nghiệp vụ nominal | N/A cho FR-12 | Chỉ giữ hợp lệ để cô lập quyền. Biên name/price thuộc FR-15, không kéo vào quyền để tạo ca. |
| Resource ID / trạng thái fixture | N/A | Định danh/membership và State Machine; không có min/max của Access Control. |
| Nguồn phiên Mobile | N/A | Native login và API replay là hai kênh bằng chứng, không phải miền số. |

## 2. Boundary Identification Table

| Boundary ID | Biến | Loại biên | Giá trị B | Tính hợp lệ / bước liền kề | SRS |
| --- | --- | --- | --- | --- | --- |
| N/A | Tất cả Input của Access Control | N/A | Không có | Không có ordered boundary được FR-12 quy định. | README.md:176-179 |

Đã hoàn thành mốc phân tích khả năng áp dụng và nhận diện biên. Phần thiết kế/coverage sẽ được bổ sung ở commit tiếp theo; chưa thực thi kiểm thử.
