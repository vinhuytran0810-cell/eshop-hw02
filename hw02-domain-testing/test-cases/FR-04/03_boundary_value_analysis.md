# Task 03 - Boundary Value Analysis: FR-04 - Personal Profile Management (Web)

Ngày lập: 2026-10-18 (Asia/Saigon). Repository: `E:/KiemThu/eshop-hw02`, origin `vinhuytran0810-cell/eshop-hw02`. Revision nền: `6ef7f1ee88a20ccc1b38c90e7ce12ba768f9df2d`. SRS: `README.md`, phiên bản 2.0. Đã đối chiếu source và cả bốn báo cáo `02_domain_testing.md`; báo cáo liên quan trực tiếp ở cùng thư mục với file này.

Phương pháp theo Task 03: 2-point chọn (L−1,L) và (U,U+1) ở biên nguyên, đúng một Valid tại biên và một Invalid ngay ngoài; 3-point chọn (B−1,B,B+1), phân Valid/Invalid theo SRS và loại trùng giá trị. Không áp công thức ±1 cho miền chưa có bước nhảy. Chỉ thay một Input Variable; các Input khác nominal, hợp lệ. Source chỉ xác định đường thao tác, không định nghĩa Expected Result. Các báo cáo Domain Testing được giữ nguyên. Chưa build/chạy ứng dụng, gửi API hoặc thực thi Test Case; không có Pass/Fail hoặc bug xác nhận.

Căn cứ: P04-01 cho cập nhật hồ sơ (`README.md:64`); P04-02 yêu cầu phone bắt đầu 0, gồm 10-11 chữ số (dòng 65); P04-03 đến P04-05 giữ email qua UI, đúng chủ hồ sơ và không tự đổi role (dòng 66-67). Source đã đọc: `frontend-web/src/pages/Profile.jsx`, `Profile()`/`handleUpdate()` dòng 5/40-64, form dòng 111-165; `frontend-web/src/context/AuthContext.jsx`, `AuthProvider()`/`login()`; `backend/server.js:100-135`, authenticateToken và callback GET/PUT /api/users/me. Regex hiện tại tại Profile.jsx:43 chỉ là quan sát SUT, không dùng làm oracle.

## 1. Khả năng áp dụng BVA cho từng Input Variable

| Input | Khả năng áp dụng | Căn cứ / giải thích |
| --- | --- | --- |
| token / phiên | N/A | Miền xác thực; không có độ dài/thời hạn do FR-04 quy định. |
| name | N/A | Không có min/max trong FR-04; không mượn giới hạn tên sản phẩm. |
| phone | Áp dụng theo độ dài L | Chuỗi bắt đầu 0, toàn chữ số, 10≤L≤11. Bước độ dài bằng một ký tự; không coi phone là số nguyên. |
| shipping_address | N/A | Không có min/max hoặc chuẩn địa chỉ trong FR-04. |
| Thao tác email trên UI | N/A | Giữ/sửa email là hành vi phân loại, không phải khoảng số. |
| Đối tượng hồ sơ | N/A | Quan hệ chủ sở hữu, không có khoảng ID nghiệp vụ. |
| Yêu cầu role | N/A | Quyền sửa role là phân loại; không tự xếp hạng số. |

## 2. Boundary Identification Table

| Boundary ID | Biến | Loại biên | Giá trị B | Tính hợp lệ / bước liền kề | SRS |
| --- | --- | --- | --- | --- | --- |
| FR04-B-L | L = độ dài phone | Dưới | 10 | Inclusive, Valid tại biên; một ký tự/chữ số mỗi bước. | P04-02, README.md:65 |
| FR04-B-U | L = độ dài phone | Trên | 11 | Inclusive, Valid tại biên; một ký tự/chữ số mỗi bước. | P04-02, README.md:65 |

Đã hoàn thành mốc phân tích khả năng áp dụng và nhận diện biên. Phần thiết kế/coverage sẽ được bổ sung ở commit tiếp theo; chưa thực thi kiểm thử.
