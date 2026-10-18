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

## Các vấn đề cần human review

1. Admin shipping→canceled: sơ đồ không có cạnh, câu chữ chỉ nói Admin mới có thể thao tác khi shipping; chưa đủ để kết luận hủy được phép.
2. Self-loop/lệnh hủy lặp canceled: trạng thái phải không đổi; chưa rõ thành công no-op hay lỗi. Không áp câu “sang trạng thái khác” để tự cấm mọi self-loop.
3. Việc hủy đơn người khác chưa nêu riêng ở FR-10; dùng đơn của chính A làm nominal, không tự dựng oracle quyền ghi từ FR-11 chỉ nói quyền xem.
4. Thông báo lỗi phải phù hợp theo O10-04 nhưng mã HTTP/nội dung nguyên văn chưa được định nghĩa.
5. Các quan sát static về shipping/canceled chỉ là rủi ro, chưa xác nhận bug; fixture và khả năng khởi chạy chưa được kiểm tra.

Đã hoàn thành đến Step 2. Các Step sau chưa được đưa vào phiên bản commit này.
