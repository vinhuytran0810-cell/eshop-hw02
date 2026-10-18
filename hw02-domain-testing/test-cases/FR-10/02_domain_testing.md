# Domain Testing cho FR-10 - Order State Machine (Web)

Ngày lập: 2026-10-18 (Asia/Saigon). SRS duy nhất: `README.md`, phiên bản 2.0. Source tham chiếu tại revision `8f1982bb8802bd6216e3c4e47da15c2e779fb6b4`. Tham khảo cấu trúc FR-04 và `hw02-domain-testing/reports/01_repository_analysis.md`; không sửa hai tài liệu này. Chỉ thiết kế, chưa chạy ứng dụng/API/Test Case; không có Pass/Fail hoặc bug được xác nhận.

| Mã yêu cầu | Nội dung SRS | Nguồn |
| --- | --- | --- |
| O10-01 | Năm trạng thái và các cạnh: Admin pending→confirmed, confirmed→shipping, shipping→delivered; User/Admin pending→canceled và confirmed→canceled. | `README.md:143-156` |
| O10-02 | delivered và canceled là trạng thái kết thúc; không chuyển sang trạng thái khác. | `README.md:160` |
| O10-03 | User không được tự hủy khi shipping; chỉ Admin mới có thể thao tác ở giai đoạn này. | `README.md:161` |
| O10-04 | Chuyển trạng thái không hợp lệ trả lỗi với thông báo phù hợp. | `README.md:162` |
| O10-AUTH | Các API Admin yêu cầu JWT hợp lệ, token role=admin. | FR-12, `README.md:176-179`; FR-02 dòng 43 |
| O10-DEP | Admin cập nhật theo FR-10; lịch sử của User chỉ hiển thị đơn chính mình. | FR-18 dòng 220-221; FR-11 dòng 166-168 |

Source đã đọc: `frontend-web/src/pages/Profile.jsx`, `fetchOrders()` dòng 14, `cancelOrder()` dòng 66, điều kiện nút dòng 200; `frontend-admin/src/App.jsx`, `fetchData()` dòng 41, `updateOrderStatus()` dòng 85, điều kiện nút dòng 814-869; `backend/server.js`, `authenticateToken()` dòng 100, callback checkout dòng 297, GET my-orders dòng 311, PUT cancel dòng 321, GET Admin orders dòng 510, PUT Admin status dòng 525; `backend/database.js:74-81`, bảng `orders`. Không có controller/module State Machine riêng. Quan sát source: nút User xét khác delivered/canceled, callback Admin có nhánh canceled→delivered. Các quan sát này không quyết định Expected Result hoặc xác nhận bug.

## Step 1 - Identify Input & Output Variables

| Mã | Input | Ý nghĩa | Ràng buộc SRS |
| --- | --- | --- | --- |
| I01 | JWT / phiên | Danh tính gửi yêu cầu có xác thực. | JWT hợp lệ; không giả định thời hạn, mã lỗi cụ thể. O10-AUTH. |
| I02 | Actor / role | User hoặc Admin thực hiện hành động. | Cạnh tiến trình do Admin; User được hủy theo sơ đồ. O10-01/O10-03/O10-AUTH. |
| I03 | Thao tác | Hủy đơn hoặc yêu cầu đặt status. | Chỉ kiểm tra hai endpoint có thật; không tự tạo endpoint xác nhận/giao riêng. |
| I04 | orderId / chủ đơn | Đơn tồn tại, có trạng thái ban đầu xác minh; dùng đơn của User A cho ca User. | ID là ngữ cảnh, không có min/max. Quy tắc hủy đơn người khác chưa phát biểu riêng trong FR-10; không suy diễn FR-11 xem đơn thành hợp đồng mọi thao tác ghi. |
| I05 | currentStatus | Trạng thái đã lưu trước request. | Năm trạng thái SRS; trạng thái kết thúc vẫn là ngữ cảnh hợp lệ, chỉ yêu cầu đổi từ đó bị cấm. |
| I06 | requestedStatus | Trạng thái đích; cancel endpoint có đích cố định canceled. | Phải thuộc năm trạng thái; tính hợp lệ còn phụ thuộc I02/I03/I05. |
| I07 | Quan hệ chuyển trạng thái | Bộ currentStatus/action/target/actor. | Phân EC kết hợp khi luật liên biến; không gọi shipping là Input luôn không hợp lệ. |

| Mã | Output | Expected Result theo SRS |
| --- | --- | --- |
| O01 | Quyết định cho phép / từ chối | Cạnh được phép và đúng role thực hiện được; cạnh cấm trả lỗi phù hợp. Không ép 200/400/403 hoặc nguyên văn thông báo. |
| O02 | Trạng thái lưu và hiển thị sau thao tác | Cạnh hợp lệ ghi đúng trạng thái đích; cạnh bị từ chối giữ trạng thái ban đầu. Đọc lại sau thao tác, không chỉ nhìn toast. |
| O03 | Tính kết thúc | Không chuyển delivered/canceled sang trạng thái khác. Cùng trạng thái cần human review. |
| O04 | Phản hồi Web và quyền | Quan sát control, phản hồi lỗi và trạng thái; UI không có nút không chứng minh API được bảo vệ. |

Endpoint xác minh: `GET /api/orders/my-orders`, `PUT /api/orders/:id/cancel`, `GET /api/admin/orders`, `PUT /api/admin/orders/:id/status`, `POST /api/checkout` (chuẩn bị đơn). Không dùng `GET /api/orders/:id` không xác thực làm kênh oracle duy nhất; đọc bằng GET của đúng chủ hoặc Admin.

## Step 2 - Identify Equivalence Classes

| EC ID | Input / quan hệ | Điều kiện | Loại | SRS | Lý do phân chia |
| --- | --- | --- | --- | --- | --- |
| FR10-EC-A01 | I01 | JWT hợp lệ. | Valid | O10-AUTH | Ngữ cảnh được xác thực. |
| FR10-EC-A02 | I01 | Thiếu JWT. | Invalid | O10-AUTH | Tách thiếu credential khỏi credential hỏng. |
| FR10-EC-A03 | I01 | JWT gửi lên không hợp lệ. | Invalid | O10-AUTH | Chọn một nguyên nhân đại diện, không nhân mọi chuỗi token. |
| FR10-EC-R01 | I02 | Actor Admin. | Valid | O10-01 | Được thực hiện các cạnh tiến trình. |
| FR10-EC-R02 | I02 | Actor User. | Valid | O10-01/O10-03 | User không luôn là invalid; hợp lệ cho hủy được phép. |
| FR10-EC-O01 | I03 | Yêu cầu đặt status qua Admin endpoint. | Valid | O10-01/O10-AUTH | Loại hành động tồn tại; quyền kiểm tra riêng. |
| FR10-EC-O02 | I03 | Yêu cầu hủy qua cancel endpoint. | Valid | O10-01/O10-03 | Đích hủy cố định. |
| FR10-EC-I01 | I04 | Đơn tồn tại; User dùng đơn chính mình, Admin dùng đơn phù hợp fixture. | Valid | O10-01/O10-DEP | ID phục vụ thiết lập trạng thái, không tự có quy tắc số học. |
| FR10-EC-C01 | I05 | currentStatus=pending. | Valid | O10-01/O10-02 | Giữ từng trạng thái để không gộp hành vi khác nhau. |
| FR10-EC-T01 | I06 | requestedStatus=pending. | Valid | O10-01 | Giá trị trạng thái được định nghĩa; quan hệ cạnh xét riêng. |
| FR10-EC-C02 | I05 | currentStatus=confirmed. | Valid | O10-01/O10-02 | Giữ từng trạng thái để không gộp hành vi khác nhau. |
| FR10-EC-T02 | I06 | requestedStatus=confirmed. | Valid | O10-01 | Giá trị trạng thái được định nghĩa; quan hệ cạnh xét riêng. |
| FR10-EC-C03 | I05 | currentStatus=shipping. | Valid | O10-01/O10-02 | Giữ từng trạng thái để không gộp hành vi khác nhau. |
| FR10-EC-T03 | I06 | requestedStatus=shipping. | Valid | O10-01 | Giá trị trạng thái được định nghĩa; quan hệ cạnh xét riêng. |
| FR10-EC-C04 | I05 | currentStatus=delivered. | Valid | O10-01/O10-02 | Giữ từng trạng thái để không gộp hành vi khác nhau. |
| FR10-EC-T04 | I06 | requestedStatus=delivered. | Valid | O10-01 | Giá trị trạng thái được định nghĩa; quan hệ cạnh xét riêng. |
| FR10-EC-C05 | I05 | currentStatus=canceled. | Valid | O10-01/O10-02 | Giữ từng trạng thái để không gộp hành vi khác nhau. |
| FR10-EC-T05 | I06 | requestedStatus=canceled. | Valid | O10-01 | Giá trị trạng thái được định nghĩa; quan hệ cạnh xét riêng. |
| FR10-EC-T06 | I06 | Nhãn đích ngoài năm trạng thái. | Invalid | O10-01/O10-04 | Sai miền nhãn, không đồng thời gán một EC cạnh sai. |
| FR10-EC-Z01 | I02/I03 | JWT User hợp lệ gọi cạnh Admin, còn cạnh trạng thái hợp lệ. | Invalid | O10-01/O10-AUTH | Chỉ quyền hành động sai. |
| FR10-EC-S01 | I07 | Admin: pending→confirmed. | Valid | O10-01 | Một cạnh sơ đồ được phép. |
| FR10-EC-S02 | I07 | Admin: pending→shipping. | Invalid | O10-01/O10-02/O10-04 | Cạnh nhảy bước/quay ngược không có trong sơ đồ. |
| FR10-EC-S03 | I07 | Admin: pending→delivered. | Invalid | O10-01/O10-02/O10-04 | Cạnh nhảy bước/quay ngược không có trong sơ đồ. |
| FR10-EC-S04 | I07 | Admin: pending→canceled. | Valid | O10-01 | Một cạnh sơ đồ được phép. |
| FR10-EC-S05 | I07 | Admin: confirmed→pending. | Invalid | O10-01/O10-02/O10-04 | Cạnh nhảy bước/quay ngược không có trong sơ đồ. |
| FR10-EC-S06 | I07 | Admin: confirmed→shipping. | Valid | O10-01 | Một cạnh sơ đồ được phép. |
| FR10-EC-S07 | I07 | Admin: confirmed→delivered. | Invalid | O10-01/O10-02/O10-04 | Cạnh nhảy bước/quay ngược không có trong sơ đồ. |
| FR10-EC-S08 | I07 | Admin: confirmed→canceled. | Valid | O10-01 | Một cạnh sơ đồ được phép. |
| FR10-EC-S09 | I07 | Admin: shipping→pending. | Invalid | O10-01/O10-02/O10-04 | Cạnh nhảy bước/quay ngược không có trong sơ đồ. |
| FR10-EC-S10 | I07 | Admin: shipping→confirmed. | Invalid | O10-01/O10-02/O10-04 | Cạnh nhảy bước/quay ngược không có trong sơ đồ. |
| FR10-EC-S11 | I07 | Admin: shipping→delivered. | Valid | O10-01 | Một cạnh sơ đồ được phép. |
| FR10-EC-S12 | I07 | Admin: delivered→pending. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-S13 | I07 | Admin: delivered→confirmed. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-S14 | I07 | Admin: delivered→shipping. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-S15 | I07 | Admin: delivered→canceled. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-S16 | I07 | Admin: canceled→pending. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-S17 | I07 | Admin: canceled→confirmed. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-S18 | I07 | Admin: canceled→shipping. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-S19 | I07 | Admin: canceled→delivered. | Invalid | O10-01/O10-02/O10-04 | Rời trạng thái kết thúc; tách từng đích để có coverage toàn ma trận. |
| FR10-EC-U01 | I07 | User hủy: pending→canceled. | Valid | O10-01/O10-02/O10-03/O10-04 | Tách theo trạng thái nguồn vì quyền hủy thay đổi. |
| FR10-EC-U02 | I07 | User hủy: confirmed→canceled. | Valid | O10-01/O10-02/O10-03/O10-04 | Tách theo trạng thái nguồn vì quyền hủy thay đổi. |
| FR10-EC-U03 | I07 | User hủy: shipping→canceled. | Invalid | O10-01/O10-02/O10-03/O10-04 | Tách theo trạng thái nguồn vì quyền hủy thay đổi. |
| FR10-EC-U04 | I07 | User hủy: delivered→canceled. | Invalid | O10-01/O10-02/O10-03/O10-04 | Tách theo trạng thái nguồn vì quyền hủy thay đổi. |

Có 43 EC. EC riêng cho từng trạng thái/đích mô tả ngữ cảnh hợp lệ; tính hợp lệ của hành động nằm ở EC quan hệ I07 hoặc quyền Z01. Mỗi ca Invalid chỉ mang một Invalid EC; không gán thêm EC trạng thái sai khi chỉ role sai. Không tạo Invalid EC cho orderId âm/không tồn tại, loại JSON, trạng thái database hỏng vì FR-10 chưa đủ hợp đồng Expected Result. Không gán EC xác định đúng/sai cho self-loop hoặc Admin shipping→canceled; các ô này được ghi Q trong ma trận để human review. Token hết hạn/thu hồi không có thời lượng đặc tả, chưa tách EC riêng.

## Step 3 - Select Best Representatives & Design Test Cases

### Preconditions và bộ bước dùng chung

A là User chủ đơn, Admin là tài khoản role=admin. T_A/T_Admin được cấp bằng POST login trong lần thực thi sau; ID đơn lấy từ dữ liệu thật, không tự giả định. Mỗi ca dùng một đơn/snapshot độc lập ở trạng thái nguồn. Chuẩn bị bằng checkout và các cạnh đã được phép; nếu bị chặn thì ghi điểm chặn, không sửa SQL/source để tạo bằng chứng. Không restart backend giữa kiểm tra vì database.js tự khởi tạo lại.

**K**: (1) xác minh actor, đơn, trạng thái nguồn và snapshot; (2) gửi đúng endpoint/header/body trong ca; (3) ghi phản hồi; (4) đọc lại bằng GET my-orders của chủ hoặc GET Admin orders; (5) đối chiếu trạng thái và thông báo.

**W-Admin**: mở tab orders, chọn đơn, thao tác control có sẵn, ghi phản hồi và request; refresh/đọc lại độc lập. **W-User**: mở /profile, thao tác Hủy đơn nếu có, ghi phản hồi và đọc lại. Không bắt một cơ chế toast/ẩn nút cụ thể khi SRS chưa quy định. Các ca UI và API giữ hai tầng coverage riêng.

### Representative Values

| EC ID | Representative Value | Lý do lựa chọn |
| --- | --- | --- |
| FR10-EC-A01 | T_Admin hoặc T_A lấy qua login. | Không tự ký bằng secret source. |
| FR10-EC-A02 | Bỏ Authorization. | Các trường khác vẫn hợp lệ. |
| FR10-EC-A03 | T_Admin sai chữ ký, giữ payload Admin. | Chỉ credential sai. |
| FR10-EC-R01 | T_Admin, role=admin. | Role đúng khi cô lập sai trạng thái. |
| FR10-EC-R02 | T_A, role=user. | Không đổi role của tài khoản. |
| FR10-EC-O01 | PUT /api/admin/orders/:id/status. | Endpoint đã đọc. |
| FR10-EC-O02 | PUT /api/orders/:id/cancel, body {}. | Không giả định endpoint hủy nhận đích bất kỳ. |
| FR10-EC-I01 | O_A xác minh thuộc User A. | Không hardcode ID. |
| FR10-EC-C01 | pending | Dùng đơn độc lập đã xác minh trạng thái. |
| FR10-EC-T01 | pending | Tách miền nhãn trạng thái khỏi cạnh hợp lệ. |
| FR10-EC-C02 | confirmed | Dùng đơn độc lập đã xác minh trạng thái. |
| FR10-EC-T02 | confirmed | Tách miền nhãn trạng thái khỏi cạnh hợp lệ. |
| FR10-EC-C03 | shipping | Dùng đơn độc lập đã xác minh trạng thái. |
| FR10-EC-T03 | shipping | Tách miền nhãn trạng thái khỏi cạnh hợp lệ. |
| FR10-EC-C04 | delivered | Dùng đơn độc lập đã xác minh trạng thái. |
| FR10-EC-T04 | delivered | Tách miền nhãn trạng thái khỏi cạnh hợp lệ. |
| FR10-EC-C05 | canceled | Dùng đơn độc lập đã xác minh trạng thái. |
| FR10-EC-T05 | canceled | Tách miền nhãn trạng thái khỏi cạnh hợp lệ. |
| FR10-EC-T06 | archived | Chuỗi ngoài danh sách, không thêm quy tắc kiểu JSON. |
| FR10-EC-Z01 | User gọi pending→confirmed. | Không chọn cạnh vốn đã bị cấm. |
| FR10-EC-S01 | pending→confirmed | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S02 | pending→shipping | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S03 | pending→delivered | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S04 | pending→canceled | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S05 | confirmed→pending | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S06 | confirmed→shipping | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S07 | confirmed→delivered | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S08 | confirmed→canceled | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S09 | shipping→pending | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S10 | shipping→confirmed | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S11 | shipping→delivered | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S12 | delivered→pending | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S13 | delivered→confirmed | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S14 | delivered→shipping | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S15 | delivered→canceled | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S16 | canceled→pending | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S17 | canceled→confirmed | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S18 | canceled→shipping | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-S19 | canceled→delivered | Một cặp nhãn xác định; actor và JWT giữ hợp lệ. |
| FR10-EC-U01 | pending→canceled bằng T_A. | JWT và chủ đơn đúng; chỉ quan hệ hủy thay đổi. |
| FR10-EC-U02 | confirmed→canceled bằng T_A. | JWT và chủ đơn đúng; chỉ quan hệ hủy thay đổi. |
| FR10-EC-U03 | shipping→canceled bằng T_A. | JWT và chủ đơn đúng; chỉ quan hệ hủy thay đổi. |
| FR10-EC-U04 | delivered→canceled bằng T_A. | JWT và chủ đơn đúng; chỉ quan hệ hủy thay đổi. |

### Domain Test Cases - Backend API

| Test Case ID | Preconditions | Inputs | Test Steps | Expected Results | ECs Covered |
| --- | --- | --- | --- | --- | --- |
| FR10-DT-K01 | Đơn O_A tồn tại ở pending; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: confirmed}; Bearer T_Admin. | Dùng K; chỉ request một lần từ pending; đọc lại bằng GET Admin orders. | Cho phép chuyển sang confirmed; đọc lại đúng confirmed. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T02, FR10-EC-S01 |
| FR10-DT-K02 | Đơn O_A tồn tại ở pending; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: shipping}; Bearer T_Admin. | Dùng K; chỉ request một lần từ pending; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn pending. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T03, FR10-EC-S02 |
| FR10-DT-K03 | Đơn O_A tồn tại ở pending; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: delivered}; Bearer T_Admin. | Dùng K; chỉ request một lần từ pending; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn pending. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T04, FR10-EC-S03 |
| FR10-DT-K04 | Đơn O_A tồn tại ở pending; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: canceled}; Bearer T_Admin. | Dùng K; chỉ request một lần từ pending; đọc lại bằng GET Admin orders. | Cho phép chuyển sang canceled; đọc lại đúng canceled. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T05, FR10-EC-S04 |
| FR10-DT-K05 | Đơn O_A tồn tại ở confirmed; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: pending}; Bearer T_Admin. | Dùng K; chỉ request một lần từ confirmed; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn confirmed. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C02, FR10-EC-T01, FR10-EC-S05 |
| FR10-DT-K06 | Đơn O_A tồn tại ở confirmed; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: shipping}; Bearer T_Admin. | Dùng K; chỉ request một lần từ confirmed; đọc lại bằng GET Admin orders. | Cho phép chuyển sang shipping; đọc lại đúng shipping. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C02, FR10-EC-T03, FR10-EC-S06 |
| FR10-DT-K07 | Đơn O_A tồn tại ở confirmed; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: delivered}; Bearer T_Admin. | Dùng K; chỉ request một lần từ confirmed; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn confirmed. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C02, FR10-EC-T04, FR10-EC-S07 |
| FR10-DT-K08 | Đơn O_A tồn tại ở confirmed; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: canceled}; Bearer T_Admin. | Dùng K; chỉ request một lần từ confirmed; đọc lại bằng GET Admin orders. | Cho phép chuyển sang canceled; đọc lại đúng canceled. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C02, FR10-EC-T05, FR10-EC-S08 |
| FR10-DT-K09 | Đơn O_A tồn tại ở shipping; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: pending}; Bearer T_Admin. | Dùng K; chỉ request một lần từ shipping; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn shipping. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C03, FR10-EC-T01, FR10-EC-S09 |
| FR10-DT-K10 | Đơn O_A tồn tại ở shipping; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: confirmed}; Bearer T_Admin. | Dùng K; chỉ request một lần từ shipping; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn shipping. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C03, FR10-EC-T02, FR10-EC-S10 |
| FR10-DT-K11 | Đơn O_A tồn tại ở shipping; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: delivered}; Bearer T_Admin. | Dùng K; chỉ request một lần từ shipping; đọc lại bằng GET Admin orders. | Cho phép chuyển sang delivered; đọc lại đúng delivered. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C03, FR10-EC-T04, FR10-EC-S11 |
| FR10-DT-K12 | Đơn O_A tồn tại ở delivered; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: pending}; Bearer T_Admin. | Dùng K; chỉ request một lần từ delivered; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn delivered. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C04, FR10-EC-T01, FR10-EC-S12 |
| FR10-DT-K13 | Đơn O_A tồn tại ở delivered; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: confirmed}; Bearer T_Admin. | Dùng K; chỉ request một lần từ delivered; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn delivered. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C04, FR10-EC-T02, FR10-EC-S13 |
| FR10-DT-K14 | Đơn O_A tồn tại ở delivered; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: shipping}; Bearer T_Admin. | Dùng K; chỉ request một lần từ delivered; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn delivered. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C04, FR10-EC-T03, FR10-EC-S14 |
| FR10-DT-K15 | Đơn O_A tồn tại ở delivered; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: canceled}; Bearer T_Admin. | Dùng K; chỉ request một lần từ delivered; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn delivered. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C04, FR10-EC-T05, FR10-EC-S15 |
| FR10-DT-K16 | Đơn O_A tồn tại ở canceled; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: pending}; Bearer T_Admin. | Dùng K; chỉ request một lần từ canceled; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn canceled. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C05, FR10-EC-T01, FR10-EC-S16 |
| FR10-DT-K17 | Đơn O_A tồn tại ở canceled; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: confirmed}; Bearer T_Admin. | Dùng K; chỉ request một lần từ canceled; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn canceled. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C05, FR10-EC-T02, FR10-EC-S17 |
| FR10-DT-K18 | Đơn O_A tồn tại ở canceled; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: shipping}; Bearer T_Admin. | Dùng K; chỉ request một lần từ canceled; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn canceled. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C05, FR10-EC-T03, FR10-EC-S18 |
| FR10-DT-K19 | Đơn O_A tồn tại ở canceled; T_Admin hợp lệ; snapshot trước thao tác. | PUT /api/admin/orders/O_A/status; {status: delivered}; Bearer T_Admin. | Dùng K; chỉ request một lần từ canceled; đọc lại bằng GET Admin orders. | Trả lỗi phù hợp, trạng thái vẫn canceled. O10-01/O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C05, FR10-EC-T04, FR10-EC-S19 |
| FR10-DT-U01 | O_A thuộc A ở pending; T_A hợp lệ. | PUT /api/orders/O_A/cancel; {}; Bearer T_A. | Dùng K; snapshot trước/sau qua GET my-orders của A. | Cho phép hủy, trạng thái canceled. O10-01. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O02, FR10-EC-I01, FR10-EC-C01, FR10-EC-T05, FR10-EC-U01 |
| FR10-DT-U02 | O_A thuộc A ở confirmed; T_A hợp lệ. | PUT /api/orders/O_A/cancel; {}; Bearer T_A. | Dùng K; snapshot trước/sau qua GET my-orders của A. | Cho phép hủy, trạng thái canceled. O10-01. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O02, FR10-EC-I01, FR10-EC-C02, FR10-EC-T05, FR10-EC-U02 |
| FR10-DT-U03 | O_A thuộc A ở shipping; T_A hợp lệ. | PUT /api/orders/O_A/cancel; {}; Bearer T_A. | Dùng K; snapshot trước/sau qua GET my-orders của A. | Từ chối hủy với lỗi phù hợp; giữ shipping. O10-02/O10-03/O10-04. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O02, FR10-EC-I01, FR10-EC-C03, FR10-EC-T05, FR10-EC-U03 |
| FR10-DT-U04 | O_A thuộc A ở delivered; T_A hợp lệ. | PUT /api/orders/O_A/cancel; {}; Bearer T_A. | Dùng K; snapshot trước/sau qua GET my-orders của A. | Từ chối hủy với lỗi phù hợp; giữ delivered. O10-02/O10-03/O10-04. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O02, FR10-EC-I01, FR10-EC-C04, FR10-EC-T05, FR10-EC-U04 |
| FR10-DT-R01 | O_A ở pending; JWT A hợp lệ, role=user. | PUT Admin status; {status:confirmed}; Bearer T_A. | Dùng K; trạng thái nguồn/đích vốn là cạnh cho phép với Admin. | Từ chối thao tác Admin của User, giữ pending; lỗi phù hợp; O10-AUTH/O10-01. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T02, FR10-EC-Z01 |
| FR10-DT-R02 | O_A ở confirmed; JWT A hợp lệ, role=user. | PUT Admin status; {status:shipping}; Bearer T_A. | Dùng K; trạng thái nguồn/đích vốn là cạnh cho phép với Admin. | Từ chối thao tác Admin của User, giữ confirmed; lỗi phù hợp; O10-AUTH/O10-01. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O01, FR10-EC-I01, FR10-EC-C02, FR10-EC-T03, FR10-EC-Z01 |
| FR10-DT-R03 | O_A ở shipping; JWT A hợp lệ, role=user. | PUT Admin status; {status:delivered}; Bearer T_A. | Dùng K; trạng thái nguồn/đích vốn là cạnh cho phép với Admin. | Từ chối thao tác Admin của User, giữ shipping; lỗi phù hợp; O10-AUTH/O10-01. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O01, FR10-EC-I01, FR10-EC-C03, FR10-EC-T04, FR10-EC-Z01 |
| FR10-DT-A01 | O_A ở pending; body confirmed; role trong payload token nếu có là admin. | PUT Admin status; {status:confirmed}; Không Authorization. | Dùng K; đối chiếu bằng một phiên Admin hợp lệ độc lập. | Không cho request cập nhật, giữ pending. O10-AUTH; không ép mã HTTP. | FR10-EC-A02, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T02 |
| FR10-DT-A02 | O_A ở pending; body confirmed; role trong payload token nếu có là admin. | PUT Admin status; {status:confirmed}; Bearer T_Admin sai chữ ký. | Dùng K; đối chiếu bằng một phiên Admin hợp lệ độc lập. | Không cho request cập nhật, giữ pending. O10-AUTH; không ép mã HTTP. | FR10-EC-A03, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T02 |
| FR10-DT-X01 | T_Admin hợp lệ; O_A ở pending. | PUT Admin status; {status:archived}. | Dùng K; chỉ nhãn đích sai. | Trả lỗi phù hợp, giữ pending. O10-01/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T06 |

### Domain Test Cases - Web Admin UI

| Test Case ID | Preconditions | Inputs | Test Steps | Expected Results | ECs Covered |
| --- | --- | --- | --- | --- | --- |
| FR10-DT-W01 | Tài khoản Admin đăng nhập; tab orders; O_A ở pending. | Thao tác yêu cầu pending→confirmed qua control gốc. | Dùng W-Admin. Nếu cạnh cấm không có control thì ghi nhận UI đã ngăn; không tự thêm nút/đổi DOM. | Thực hiện được cạnh pending→confirmed; đọc lại đúng confirmed. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T02, FR10-EC-S01 |
| FR10-DT-W02 | Tài khoản Admin đăng nhập; tab orders; O_A ở pending. | Thao tác yêu cầu pending→canceled qua control gốc. | Dùng W-Admin. Nếu cạnh cấm không có control thì ghi nhận UI đã ngăn; không tự thêm nút/đổi DOM. | Thực hiện được cạnh pending→canceled; đọc lại đúng canceled. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C01, FR10-EC-T05, FR10-EC-S04 |
| FR10-DT-W03 | Tài khoản Admin đăng nhập; tab orders; O_A ở confirmed. | Thao tác yêu cầu confirmed→shipping qua control gốc. | Dùng W-Admin. Nếu cạnh cấm không có control thì ghi nhận UI đã ngăn; không tự thêm nút/đổi DOM. | Thực hiện được cạnh confirmed→shipping; đọc lại đúng shipping. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C02, FR10-EC-T03, FR10-EC-S06 |
| FR10-DT-W04 | Tài khoản Admin đăng nhập; tab orders; O_A ở confirmed. | Thao tác yêu cầu confirmed→canceled qua control gốc. | Dùng W-Admin. Nếu cạnh cấm không có control thì ghi nhận UI đã ngăn; không tự thêm nút/đổi DOM. | Thực hiện được cạnh confirmed→canceled; đọc lại đúng canceled. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C02, FR10-EC-T05, FR10-EC-S08 |
| FR10-DT-W05 | Tài khoản Admin đăng nhập; tab orders; O_A ở shipping. | Thao tác yêu cầu shipping→delivered qua control gốc. | Dùng W-Admin. Nếu cạnh cấm không có control thì ghi nhận UI đã ngăn; không tự thêm nút/đổi DOM. | Thực hiện được cạnh shipping→delivered; đọc lại đúng delivered. O10-01. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C03, FR10-EC-T04, FR10-EC-S11 |
| FR10-DT-W06 | Tài khoản Admin đăng nhập; tab orders; O_A ở canceled. | Thao tác yêu cầu canceled→delivered qua control gốc. | Dùng W-Admin. Nếu cạnh cấm không có control thì ghi nhận UI đã ngăn; không tự thêm nút/đổi DOM. | Không cho canceled chuyển sang delivered; giữ canceled. Nếu có control và thao tác tới backend, phải nhận lỗi phù hợp. O10-02/O10-04. | FR10-EC-A01, FR10-EC-R01, FR10-EC-O01, FR10-EC-I01, FR10-EC-C05, FR10-EC-T04, FR10-EC-S19 |

### Domain Test Cases - Web UI

| Test Case ID | Preconditions | Inputs | Test Steps | Expected Results | ECs Covered |
| --- | --- | --- | --- | --- | --- |
| FR10-DT-W07 | User A đã đăng nhập; /profile; O_A ở pending. | Thử Hủy đơn qua control gốc. | Dùng W-User; nếu UI không có nút thì ghi nhận giới hạn tại UI, không chuyển thành ca API. | Cho phép hủy, đọc lại canceled. O10-01. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O02, FR10-EC-I01, FR10-EC-C01, FR10-EC-T05, FR10-EC-U01 |
| FR10-DT-W08 | User A đã đăng nhập; /profile; O_A ở confirmed. | Thử Hủy đơn qua control gốc. | Dùng W-User; nếu UI không có nút thì ghi nhận giới hạn tại UI, không chuyển thành ca API. | Cho phép hủy, đọc lại canceled. O10-01. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O02, FR10-EC-I01, FR10-EC-C02, FR10-EC-T05, FR10-EC-U02 |
| FR10-DT-W09 | User A đã đăng nhập; /profile; O_A ở shipping. | Thử Hủy đơn qua control gốc. | Dùng W-User; nếu UI không có nút thì ghi nhận giới hạn tại UI, không chuyển thành ca API. | Không cho User tự hủy shipping; giữ shipping. Nếu request tới backend phải bị từ chối phù hợp. O10-03/O10-04. | FR10-EC-A01, FR10-EC-R02, FR10-EC-O02, FR10-EC-I01, FR10-EC-C03, FR10-EC-T05, FR10-EC-U03 |

### EC Coverage Matrix

| EC ID | Test Cases được thiết kế | Phạm vi |
| --- | --- | --- |
| FR10-EC-A01 | FR10-DT-K01, FR10-DT-K02, FR10-DT-K03, FR10-DT-K04, FR10-DT-K05, FR10-DT-K06, FR10-DT-K07, FR10-DT-K08, FR10-DT-K09, FR10-DT-K10, FR10-DT-K11, FR10-DT-K12, FR10-DT-K13, FR10-DT-K14, FR10-DT-K15, FR10-DT-K16, FR10-DT-K17, FR10-DT-K18, FR10-DT-K19, FR10-DT-U01, FR10-DT-U02, FR10-DT-U03, FR10-DT-U04, FR10-DT-R01, FR10-DT-R02, FR10-DT-R03, FR10-DT-X01, FR10-DT-W01, FR10-DT-W02, FR10-DT-W03, FR10-DT-W04, FR10-DT-W05, FR10-DT-W06, FR10-DT-W07, FR10-DT-W08, FR10-DT-W09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-A02 | FR10-DT-A01 | Coverage thiết kế; chưa thực thi |
| FR10-EC-A03 | FR10-DT-A02 | Coverage thiết kế; chưa thực thi |
| FR10-EC-R01 | FR10-DT-K01, FR10-DT-K02, FR10-DT-K03, FR10-DT-K04, FR10-DT-K05, FR10-DT-K06, FR10-DT-K07, FR10-DT-K08, FR10-DT-K09, FR10-DT-K10, FR10-DT-K11, FR10-DT-K12, FR10-DT-K13, FR10-DT-K14, FR10-DT-K15, FR10-DT-K16, FR10-DT-K17, FR10-DT-K18, FR10-DT-K19, FR10-DT-A01, FR10-DT-A02, FR10-DT-X01, FR10-DT-W01, FR10-DT-W02, FR10-DT-W03, FR10-DT-W04, FR10-DT-W05, FR10-DT-W06 | Coverage thiết kế; chưa thực thi |
| FR10-EC-R02 | FR10-DT-U01, FR10-DT-U02, FR10-DT-U03, FR10-DT-U04, FR10-DT-R01, FR10-DT-R02, FR10-DT-R03, FR10-DT-W07, FR10-DT-W08, FR10-DT-W09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-O01 | FR10-DT-K01, FR10-DT-K02, FR10-DT-K03, FR10-DT-K04, FR10-DT-K05, FR10-DT-K06, FR10-DT-K07, FR10-DT-K08, FR10-DT-K09, FR10-DT-K10, FR10-DT-K11, FR10-DT-K12, FR10-DT-K13, FR10-DT-K14, FR10-DT-K15, FR10-DT-K16, FR10-DT-K17, FR10-DT-K18, FR10-DT-K19, FR10-DT-R01, FR10-DT-R02, FR10-DT-R03, FR10-DT-A01, FR10-DT-A02, FR10-DT-X01, FR10-DT-W01, FR10-DT-W02, FR10-DT-W03, FR10-DT-W04, FR10-DT-W05, FR10-DT-W06 | Coverage thiết kế; chưa thực thi |
| FR10-EC-O02 | FR10-DT-U01, FR10-DT-U02, FR10-DT-U03, FR10-DT-U04, FR10-DT-W07, FR10-DT-W08, FR10-DT-W09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-I01 | FR10-DT-K01, FR10-DT-K02, FR10-DT-K03, FR10-DT-K04, FR10-DT-K05, FR10-DT-K06, FR10-DT-K07, FR10-DT-K08, FR10-DT-K09, FR10-DT-K10, FR10-DT-K11, FR10-DT-K12, FR10-DT-K13, FR10-DT-K14, FR10-DT-K15, FR10-DT-K16, FR10-DT-K17, FR10-DT-K18, FR10-DT-K19, FR10-DT-U01, FR10-DT-U02, FR10-DT-U03, FR10-DT-U04, FR10-DT-R01, FR10-DT-R02, FR10-DT-R03, FR10-DT-A01, FR10-DT-A02, FR10-DT-X01, FR10-DT-W01, FR10-DT-W02, FR10-DT-W03, FR10-DT-W04, FR10-DT-W05, FR10-DT-W06, FR10-DT-W07, FR10-DT-W08, FR10-DT-W09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-C01 | FR10-DT-K01, FR10-DT-K02, FR10-DT-K03, FR10-DT-K04, FR10-DT-U01, FR10-DT-R01, FR10-DT-A01, FR10-DT-A02, FR10-DT-X01, FR10-DT-W01, FR10-DT-W02, FR10-DT-W07 | Coverage thiết kế; chưa thực thi |
| FR10-EC-T01 | FR10-DT-K05, FR10-DT-K09, FR10-DT-K12, FR10-DT-K16 | Coverage thiết kế; chưa thực thi |
| FR10-EC-C02 | FR10-DT-K05, FR10-DT-K06, FR10-DT-K07, FR10-DT-K08, FR10-DT-U02, FR10-DT-R02, FR10-DT-W03, FR10-DT-W04, FR10-DT-W08 | Coverage thiết kế; chưa thực thi |
| FR10-EC-T02 | FR10-DT-K01, FR10-DT-K10, FR10-DT-K13, FR10-DT-K17, FR10-DT-R01, FR10-DT-A01, FR10-DT-A02, FR10-DT-W01 | Coverage thiết kế; chưa thực thi |
| FR10-EC-C03 | FR10-DT-K09, FR10-DT-K10, FR10-DT-K11, FR10-DT-U03, FR10-DT-R03, FR10-DT-W05, FR10-DT-W09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-T03 | FR10-DT-K02, FR10-DT-K06, FR10-DT-K14, FR10-DT-K18, FR10-DT-R02, FR10-DT-W03 | Coverage thiết kế; chưa thực thi |
| FR10-EC-C04 | FR10-DT-K12, FR10-DT-K13, FR10-DT-K14, FR10-DT-K15, FR10-DT-U04 | Coverage thiết kế; chưa thực thi |
| FR10-EC-T04 | FR10-DT-K03, FR10-DT-K07, FR10-DT-K11, FR10-DT-K19, FR10-DT-R03, FR10-DT-W05, FR10-DT-W06 | Coverage thiết kế; chưa thực thi |
| FR10-EC-C05 | FR10-DT-K16, FR10-DT-K17, FR10-DT-K18, FR10-DT-K19, FR10-DT-W06 | Coverage thiết kế; chưa thực thi |
| FR10-EC-T05 | FR10-DT-K04, FR10-DT-K08, FR10-DT-K15, FR10-DT-U01, FR10-DT-U02, FR10-DT-U03, FR10-DT-U04, FR10-DT-W02, FR10-DT-W04, FR10-DT-W07, FR10-DT-W08, FR10-DT-W09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-T06 | FR10-DT-X01 | Coverage thiết kế; chưa thực thi |
| FR10-EC-Z01 | FR10-DT-R01, FR10-DT-R02, FR10-DT-R03 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S01 | FR10-DT-K01, FR10-DT-W01 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S02 | FR10-DT-K02 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S03 | FR10-DT-K03 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S04 | FR10-DT-K04, FR10-DT-W02 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S05 | FR10-DT-K05 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S06 | FR10-DT-K06, FR10-DT-W03 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S07 | FR10-DT-K07 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S08 | FR10-DT-K08, FR10-DT-W04 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S09 | FR10-DT-K09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S10 | FR10-DT-K10 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S11 | FR10-DT-K11, FR10-DT-W05 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S12 | FR10-DT-K12 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S13 | FR10-DT-K13 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S14 | FR10-DT-K14 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S15 | FR10-DT-K15 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S16 | FR10-DT-K16 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S17 | FR10-DT-K17 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S18 | FR10-DT-K18 | Coverage thiết kế; chưa thực thi |
| FR10-EC-S19 | FR10-DT-K19, FR10-DT-W06 | Coverage thiết kế; chưa thực thi |
| FR10-EC-U01 | FR10-DT-U01, FR10-DT-W07 | Coverage thiết kế; chưa thực thi |
| FR10-EC-U02 | FR10-DT-U02, FR10-DT-W08 | Coverage thiết kế; chưa thực thi |
| FR10-EC-U03 | FR10-DT-U03, FR10-DT-W09 | Coverage thiết kế; chưa thực thi |
| FR10-EC-U04 | FR10-DT-U04 | Coverage thiết kế; chưa thực thi |

Thiết kế 38 ca. Các cạnh sai khác nhau được giữ riêng để đáp ứng ma trận đầy đủ, không nhân tổ hợp JWT lỗi cho mọi cạnh. Ba ca R01-R03 tách quyền cho ba hành động Admin với cạnh vốn hợp lệ; các ca chuyển sai giữ JWT Admin đúng. UI bổ sung khả năng thao tác/control hiển thị; không được dùng UI không có nút thay bằng chứng API từ chối. Coverage trên là coverage thiết kế, không phải kết quả chạy.

### State Transition Coverage Matrix - Admin

Ký hiệu: V = cạnh hợp lệ; I = cạnh cấm; Q = cần làm rõ, không đưa Expected Result tự suy diễn. Mỗi ô V/I liên kết ca API; các ca UI bổ sung nằm ở bảng Test Cases.

| Nguồn \ Đích | pending | confirmed | shipping | delivered | canceled |
| --- | --- | --- | --- | --- | --- |
| pending | Q | V FR10-DT-K01 | I FR10-DT-K02 | I FR10-DT-K03 | V FR10-DT-K04 |
| confirmed | I FR10-DT-K05 | Q | V FR10-DT-K06 | I FR10-DT-K07 | V FR10-DT-K08 |
| shipping | I FR10-DT-K09 | I FR10-DT-K10 | Q | V FR10-DT-K11 | Q |
| delivered | I FR10-DT-K12 | I FR10-DT-K13 | I FR10-DT-K14 | Q | I FR10-DT-K15 |
| canceled | I FR10-DT-K16 | I FR10-DT-K17 | I FR10-DT-K18 | I FR10-DT-K19 | Q |

Có 25 ô: 5 cạnh V, 14 cạnh I và 6 ô Q (5 self-loop, Admin shipping→canceled). 19/19 cạnh có oracle rõ được liên kết Test Case; không báo 25/25 đã kiểm thử.

### State Transition Coverage Matrix - User

| Trạng thái nguồn | Hủy → canceled | Tự xác nhận/giao/hoàn tất |
| --- | --- | --- |
| pending | V FR10-DT-U01 | I: FR10-DT-R01 (pending→confirmed) |
| confirmed | V FR10-DT-U02 | I: FR10-DT-R02 (confirmed→shipping) |
| shipping | I FR10-DT-U03 | I: FR10-DT-R03 (shipping→delivered) |
| delivered | I FR10-DT-U04 | Không có hành động tiến trình hợp lệ từ final state; không cần ghép đồng thời sai role và sai cạnh. |
| canceled | Q: canceled→canceled là lệnh lặp, chưa có hợp đồng phản hồi. | Không có hành động tiến trình hợp lệ từ final state; không cần ghép đồng thời sai role và sai cạnh. |

Các cạnh đích khác cho User qua Admin endpoint bị lớp quyền Z01 ngăn; không tạo 25 ca User làm lặp lớp quyền. Ma trận Admin đã bao phủ toàn bộ cặp nguồn/đích có oracle; ma trận User bổ sung phạm vi quyền và hủy.

## Các vấn đề cần human review

1. Admin shipping→canceled: sơ đồ không có cạnh, câu chữ chỉ nói Admin mới có thể thao tác khi shipping; chưa đủ để kết luận hủy được phép.
2. Self-loop/lệnh hủy lặp canceled: trạng thái phải không đổi; chưa rõ thành công no-op hay lỗi. Không áp câu “sang trạng thái khác” để tự cấm mọi self-loop.
3. Việc hủy đơn người khác chưa nêu riêng ở FR-10; dùng đơn của chính A làm nominal, không tự dựng oracle quyền ghi từ FR-11 chỉ nói quyền xem.
4. Thông báo lỗi phải phù hợp theo O10-04 nhưng mã HTTP/nội dung nguyên văn chưa được định nghĩa.
5. Các quan sát static về shipping/canceled chỉ là rủi ro, chưa xác nhận bug; fixture và khả năng khởi chạy chưa được kiểm tra.

Đã hoàn thành đến Step 3. Các Step sau chưa được đưa vào phiên bản commit này.
