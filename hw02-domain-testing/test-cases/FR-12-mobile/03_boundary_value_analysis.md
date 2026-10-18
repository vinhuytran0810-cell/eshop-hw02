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

## 3. Bảng 2-point BVA và 3-point BVA

### 2-point BVA

| Biên | Bộ giá trị | Valid/Invalid | Kết luận |
| --- | --- | --- | --- |
| N/A | Không có | N/A | Không áp dụng; không tạo giá trị/Test Case giả. |

### 3-point BVA

| Biên | Bộ giá trị | Valid/Invalid | Kết luận |
| --- | --- | --- | --- |
| N/A | Không có | N/A | Không áp dụng; không tạo giá trị/Test Case giả. |

JWT hợp lệ AND role=admin tạo ranh giới logic cho phép/từ chối, không phải BVA số học. Không tự tạo JWT dài N/N+1, token hết hạn sau 30 giây/1 giờ, giới hạn số quyền hoặc thứ tự endpoint. Expired token nếu có thuộc EC không hợp lệ trong Domain Testing, không tạo phép thử thời gian khi SRS chưa quy định lifetime.

Mobile UI chỉ có login khách hàng; không có control Admin để kiểm thử trực tiếp. Domain Testing đã tách 2 ca native login chuẩn bị phiên và 52 ca Backend API replay theo token phiên Mobile. BVA N/A ở cả native UI và API authorization; không đổi nhãn các ca này thành BVA. Nếu kiểm thử API sau này vẫn cần provenance token từ native login, không dùng API login trực tiếp rồi gọi đó là Mobile UI.

## 4. BVA Test Cases

**N/A - không có BVA Test Cases.** Không gán ID giả, không chạy ca Domain Testing/State Transition dưới tên BVA; 0 ca là kết quả phân tích có căn cứ, không phải thiếu thiết kế.

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| N/A | N/A | N/A | Không thực thi | BVA không áp dụng theo SRS hiện tại | N/A |

## 5. Boundary Coverage Matrix

| Input / phạm vi | 2-point | 3-point | Test Cases | Kết luận |
| --- | --- | --- | --- | --- |
| JWT / Authorization | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| role claim | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| Protected method/route | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| HTTP method | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| Body nghiệp vụ nominal | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| Resource ID / trạng thái fixture | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |
| Nguồn phiên Mobile | N/A | N/A | Không có | Không có boundary số học xác định theo SRS. |

Coverage BVA là N/A, không ghi 100% hoặc giả định có boundary được kiểm thử.

## 6. Giả định, giới hạn và các điểm cần human review

1. BVA N/A cho FR-12 theo SRS hiện tại; Domain Testing vẫn cần kiểm tra JWT, role và từng protected API.
2. Không diễn giải việc Mobile không có Admin UI thành bằng chứng server bảo vệ quyền. Không sửa/thêm màn hình hoặc API_URL để tạo BVA.
3. SRS chưa có JWT lifetime, revocation sau logout, clock skew, độ dài token hay số lượng quyền tối đa. Không tự đặt biên theo hiện trạng source.
4. IP LAN hardcode có reachable hay không và khả năng capture native token là Preconditions cho thực thi sau, không phải boundary nghiệp vụ.
5. Các biên payload của FR-15 được báo cáo riêng. Không nhập chúng vào BVA quyền để tăng số ca.

Đã hoàn thành thiết kế Task 03 cho chức năng này. Dừng để human review; chưa chuyển sang Test Execution.
