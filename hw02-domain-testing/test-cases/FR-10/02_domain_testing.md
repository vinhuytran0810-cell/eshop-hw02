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

## Các vấn đề cần human review

1. Admin shipping→canceled: sơ đồ không có cạnh, câu chữ chỉ nói Admin mới có thể thao tác khi shipping; chưa đủ để kết luận hủy được phép.
2. Self-loop/lệnh hủy lặp canceled: trạng thái phải không đổi; chưa rõ thành công no-op hay lỗi. Không áp câu “sang trạng thái khác” để tự cấm mọi self-loop.
3. Việc hủy đơn người khác chưa nêu riêng ở FR-10; dùng đơn của chính A làm nominal, không tự dựng oracle quyền ghi từ FR-11 chỉ nói quyền xem.
4. Thông báo lỗi phải phù hợp theo O10-04 nhưng mã HTTP/nội dung nguyên văn chưa được định nghĩa.
5. Các quan sát static về shipping/canceled chỉ là rủi ro, chưa xác nhận bug; fixture và khả năng khởi chạy chưa được kiểm tra.

Đã hoàn thành đến Step 1. Các Step sau chưa được đưa vào phiên bản commit này.
