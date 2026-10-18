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

## 3. Bảng 2-point BVA và 3-point BVA

### 2-point BVA

| Boundary ID | 2-point BVA | Phân loại | Dữ liệu / giới hạn |
| --- | --- | --- | --- |
| FR04-B-L | (9, 10) | 9: Invalid; 10: Valid | L=9: 012345678; L=10: 0123456789 |
| FR04-B-U | (11, 12) | 11: Valid; 12: Invalid | L=11: 01234567890; L=12: 012345678901 |

### 3-point BVA

| Boundary ID | 3-point BVA | Phân loại | Dữ liệu / giới hạn |
| --- | --- | --- | --- |
| FR04-B-L | (9, 10, 11) | 9: Invalid; 10: Valid; 11: Valid | L=9: 012345678; L=10: 0123456789; L=11: 01234567890 |
| FR04-B-U | (10, 11, 12) | 10: Valid; 11: Valid; 12: Invalid | L=10: 0123456789; L=11: 01234567890; L=12: 012345678901 |

Dataset sau loại trùng của cả 2-point và 3-point đều là {9,10,11,12}. Giữ 4 giá trị ở mỗi tầng, tổng 8 Test Cases vật lý; không tạo thêm ca riêng cho từng phương pháp hoặc nhân đôi 10/11 giữa hai bộ 3-point. Do khoảng hợp lệ chỉ có độ dài 10 và 11, không có độ dài nguyên ở giữa để làm nominal nội miền; chọn một trong hai độ dài hợp lệ khi cần nominal, không dùng 10.5. Prefix 0 là điều kiện phân lớp giữ cố định, không được tính như một Boundary Value khác.

## 4. BVA Test Cases

A là tài khoản User hợp lệ; T_A cấp từ POST /api/login ở lần thực thi sau. Snapshot hồ sơ A trước mỗi ca; B là tài khoản khác dùng làm đối chứng nếu cần. Giữ `name = Nguyễn Minh An`, `shipping_address = 25 Nguyễn Trãi, Phường Bến Thành, TP.HCM`; email giữ nguyên qua UI; không gửi role hoặc định danh người khác; target luôn là A qua /me. Các nominal này phải hợp lệ và độc lập với phone; snapshot và giá trị mới chuẩn bị khi thực thi, chưa tạo trong nhiệm vụ này.

Phone(L) là L ký tự đầu của chuỗi `01234567890123456789`. Các giá trị chọn đều có số đầu 0, còn lại chỉ là chữ số; chỉ L thay đổi. Hai tầng dùng cùng dataset nhưng khác điểm đi vào, không thay thế bằng chứng cho nhau.

### Web UI

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| FR04-BVA-W-L9 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 012345678; L=9; các Input còn lại giữ nominal. | (1) Mở /profile; (2) chỉ thay phone bằng giá trị chỉ định, tên/địa chỉ giữ nominal; (3) bấm Cập nhật qua control gốc; (4) ghi UI/request thực sự xảy ra; (5) tải lại/GET /api/users/me bằng T_A để đối chiếu. Không chuyển thành ca API nếu UI chặn. | Không ghi đè phone bằng chuỗi sai độ dài như phone hợp lệ. Không ép rollback tất cả trường khác, mã HTTP hoặc câu thông báo. Email UI/role/phạm vi hồ sơ vẫn được bảo vệ. P04-02 đến P04-05. | FR04-B-L/2-point/dưới; FR04-B-L/3-point/dưới; EC Domain: FR04-EC-P-03 |
| FR04-BVA-W-L10 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 0123456789; L=10; các Input còn lại giữ nominal. | (1) Mở /profile; (2) chỉ thay phone bằng giá trị chỉ định, tên/địa chỉ giữ nominal; (3) bấm Cập nhật qua control gốc; (4) ghi UI/request thực sự xảy ra; (5) tải lại/GET /api/users/me bằng T_A để đối chiếu. Không chuyển thành ca API nếu UI chặn. | Cho phép cập nhật; phone đọc lại đúng chuỗi gửi, giữ số 0 đầu. Hồ sơ A có dữ liệu hợp lệ đã gửi; không tự đổi role, không tác động hồ sơ khác. P04-01/P04-02/P04-04/P04-05. | FR04-B-L/2-point/tại biên; FR04-B-L/3-point/tại biên; FR04-B-U/3-point/dưới; EC Domain: FR04-EC-P-01 |
| FR04-BVA-W-L11 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 01234567890; L=11; các Input còn lại giữ nominal. | (1) Mở /profile; (2) chỉ thay phone bằng giá trị chỉ định, tên/địa chỉ giữ nominal; (3) bấm Cập nhật qua control gốc; (4) ghi UI/request thực sự xảy ra; (5) tải lại/GET /api/users/me bằng T_A để đối chiếu. Không chuyển thành ca API nếu UI chặn. | Cho phép cập nhật; phone đọc lại đúng chuỗi gửi, giữ số 0 đầu. Hồ sơ A có dữ liệu hợp lệ đã gửi; không tự đổi role, không tác động hồ sơ khác. P04-01/P04-02/P04-04/P04-05. | FR04-B-L/3-point/trên; FR04-B-U/2-point/tại biên; FR04-B-U/3-point/tại biên; EC Domain: FR04-EC-P-02 |
| FR04-BVA-W-L12 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 012345678901; L=12; các Input còn lại giữ nominal. | (1) Mở /profile; (2) chỉ thay phone bằng giá trị chỉ định, tên/địa chỉ giữ nominal; (3) bấm Cập nhật qua control gốc; (4) ghi UI/request thực sự xảy ra; (5) tải lại/GET /api/users/me bằng T_A để đối chiếu. Không chuyển thành ca API nếu UI chặn. | Không ghi đè phone bằng chuỗi sai độ dài như phone hợp lệ. Không ép rollback tất cả trường khác, mã HTTP hoặc câu thông báo. Email UI/role/phạm vi hồ sơ vẫn được bảo vệ. P04-02 đến P04-05. | FR04-B-U/2-point/trên; FR04-B-U/3-point/trên; EC Domain: FR04-EC-P-04 |

### Backend API

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| FR04-BVA-K-L9 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 012345678; L=9; các Input còn lại giữ nominal. | (1) Snapshot A; (2) PUT /api/users/me, Bearer T_A, JSON {name: nominal, phone: giá trị chỉ định, shipping_address: nominal}; (3) ghi response; (4) GET /api/users/me để đọc lại; không đi qua validation UI. | Không ghi đè phone bằng chuỗi sai độ dài như phone hợp lệ. Không ép rollback tất cả trường khác, mã HTTP hoặc câu thông báo. Email UI/role/phạm vi hồ sơ vẫn được bảo vệ. P04-02 đến P04-05. | FR04-B-L/2-point/dưới; FR04-B-L/3-point/dưới; EC Domain: FR04-EC-P-03 |
| FR04-BVA-K-L10 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 0123456789; L=10; các Input còn lại giữ nominal. | (1) Snapshot A; (2) PUT /api/users/me, Bearer T_A, JSON {name: nominal, phone: giá trị chỉ định, shipping_address: nominal}; (3) ghi response; (4) GET /api/users/me để đọc lại; không đi qua validation UI. | Cho phép cập nhật; phone đọc lại đúng chuỗi gửi, giữ số 0 đầu. Hồ sơ A có dữ liệu hợp lệ đã gửi; không tự đổi role, không tác động hồ sơ khác. P04-01/P04-02/P04-04/P04-05. | FR04-B-L/2-point/tại biên; FR04-B-L/3-point/tại biên; FR04-B-U/3-point/dưới; EC Domain: FR04-EC-P-01 |
| FR04-BVA-K-L11 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 01234567890; L=11; các Input còn lại giữ nominal. | (1) Snapshot A; (2) PUT /api/users/me, Bearer T_A, JSON {name: nominal, phone: giá trị chỉ định, shipping_address: nominal}; (3) ghi response; (4) GET /api/users/me để đọc lại; không đi qua validation UI. | Cho phép cập nhật; phone đọc lại đúng chuỗi gửi, giữ số 0 đầu. Hồ sơ A có dữ liệu hợp lệ đã gửi; không tự đổi role, không tác động hồ sơ khác. P04-01/P04-02/P04-04/P04-05. | FR04-B-L/3-point/trên; FR04-B-U/2-point/tại biên; FR04-B-U/3-point/tại biên; EC Domain: FR04-EC-P-02 |
| FR04-BVA-K-L12 | A đã đăng nhập / T_A hợp lệ; nominal và snapshot theo mục 4; không phụ thuộc ca trước. | phone = 012345678901; L=12; các Input còn lại giữ nominal. | (1) Snapshot A; (2) PUT /api/users/me, Bearer T_A, JSON {name: nominal, phone: giá trị chỉ định, shipping_address: nominal}; (3) ghi response; (4) GET /api/users/me để đọc lại; không đi qua validation UI. | Không ghi đè phone bằng chuỗi sai độ dài như phone hợp lệ. Không ép rollback tất cả trường khác, mã HTTP hoặc câu thông báo. Email UI/role/phạm vi hồ sơ vẫn được bảo vệ. P04-02 đến P04-05. | FR04-B-U/2-point/trên; FR04-B-U/3-point/trên; EC Domain: FR04-EC-P-04 |

## 5. Boundary Coverage Matrix

| Boundary ID | Phương pháp / tầng | Giá trị cần bao phủ | Test Cases dùng chung | Trạng thái coverage |
| --- | --- | --- | --- | --- |
| FR04-B-L | 2-point / Web UI | 9 Invalid; 10 Valid | FR04-BVA-W-L9, FR04-BVA-W-L10 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR04-B-L | 2-point / Backend API | 9 Invalid; 10 Valid | FR04-BVA-K-L9, FR04-BVA-K-L10 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR04-B-L | 3-point / Web UI | 9 Invalid; 10 Valid; 11 Valid | FR04-BVA-W-L9, FR04-BVA-W-L10, FR04-BVA-W-L11 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR04-B-L | 3-point / Backend API | 9 Invalid; 10 Valid; 11 Valid | FR04-BVA-K-L9, FR04-BVA-K-L10, FR04-BVA-K-L11 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR04-B-U | 2-point / Web UI | 11 Valid; 12 Invalid | FR04-BVA-W-L11, FR04-BVA-W-L12 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR04-B-U | 2-point / Backend API | 11 Valid; 12 Invalid | FR04-BVA-K-L11, FR04-BVA-K-L12 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR04-B-U | 3-point / Web UI | 10 Valid; 11 Valid; 12 Invalid | FR04-BVA-W-L10, FR04-BVA-W-L11, FR04-BVA-W-L12 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR04-B-U | 3-point / Backend API | 10 Valid; 11 Valid; 12 Invalid | FR04-BVA-K-L10, FR04-BVA-K-L11, FR04-BVA-K-L12 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |

Các ID trong matrix tham chiếu đúng một dòng Test Case vật lý. Không cộng trùng Test Case giữa 2-point và 3-point; số ca đã thực thi là 0.

## 6. Giả định, giới hạn và các điểm cần human review

1. Độ dài rỗng/phone bỏ trường/null và chính sách xóa phone chưa rõ trong SRS; nằm ngoài các biên 10/11 được giao, không tự thêm bộ required.
2. Không có giới hạn name/address để áp BVA. Không bổ sung chuẩn đầu số nhà mạng ngoài yêu cầu bắt đầu 0 và 10-11 chữ số.
3. Không đổi regex/code để đi qua UI. Nếu ca bị chặn bởi UI, ghi bằng chứng thực tế khi chạy sau; ca API độc lập kiểm tra backend.
4. Các ca được dùng chung giữa hai phương pháp; cùng một giá trị ở tầng UI và API vẫn là hai mục tiêu khác nhau. Coverage là thiết kế, chưa thực thi.

Đã hoàn thành thiết kế Task 03 cho chức năng này. Dừng để human review; chưa chuyển sang Test Execution.
