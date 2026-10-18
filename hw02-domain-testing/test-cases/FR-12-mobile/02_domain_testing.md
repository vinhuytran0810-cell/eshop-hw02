# Domain Testing cho FR-12 - Access Control (Pool D - Mobile)

Ngày lập: 2026-10-18 (Asia/Saigon). Repository `E:/KiemThu/eshop-hw02`, origin `vinhuytran0810-cell/eshop-hw02`; source revision `8f1982bb8802bd6216e3c4e47da15c2e779fb6b4`. Chỉ thiết kế, chưa khởi chạy Mobile/backend, capture traffic, gửi API hoặc thực thi Test Case; không có Pass/Fail hay bug xác nhận. SRS duy nhất là `README.md`; báo cáo repository và FR-04 chỉ định hướng/cấu trúc.

| Mã yêu cầu | Business rule giữ nguyên từ SRS | Nguồn |
| --- | --- | --- |
| A12-01 | Phân hệ Admin chỉ dành cho tài khoản role=admin. Không suy ra toàn bộ ứng dụng Mobile chỉ cho Admin. | `README.md:176` |
| A12-02 | Tất cả /api/admin/* yêu cầu JWT hợp lệ và role=admin trong token. | `README.md:177-179` |
| A12-03 | Các API mutation POST/PUT/DELETE của products/categories/coupons yêu cầu JWT hợp lệ và role=admin trong token. | `README.md:177-179` |
| A12-DEP | Đăng nhập cấp JWT; client gửi Bearer khi có xác thực. Mobile có các chức năng khách hàng gồm login/logout/profile/history. | FR-02 dòng 43; FR-20 dòng 235; SEC-03 dòng 280 củng cố kiểm tra role |

### Triển khai và giới hạn Mobile đã xác minh

`frontend-mobile/index.js` đăng ký `App`; toàn bộ màn hình ở `frontend-mobile/App.js`. API_URL dòng 16 là http://192.168.10.13:3000/api; token/user giữ React state dòng 34-35. `handleLogin()` dòng 186-207 cấp state từ POST login, không phân nhánh role; `logout()` dòng 161-169 xóa state cục bộ. `renderLogin()` dòng 759, `renderHeader()` dòng 422, `renderProfile()` dòng 893. Điều kiện view dòng 978-985 chỉ có cart/checkout/login/register/forgotPassword/profile/productDetail và Home mặc định. Không có Admin screen, Admin navigation hay lời gọi /admin/*, product/category/coupon CRUD mutations từ Mobile.

Backend: `backend/server.js`, callback login dòng 32, jwt.sign dòng 51; `authenticateToken()` dòng 100-110 chỉ verify JWT, không kiểm tra role; các callback route dưới bảng. `backend/database.js` có users/products/categories/orders/coupons phục vụ snapshot; không có ORM model riêng. Đây là quan sát static, không thay thế quy tắc đúng hoặc xác nhận quyền truy cập thực tế.

| Route ID | Method | API đã xác minh | Callback server.js dòng | Quan sát protection source |
| --- | --- | --- | --- | --- |
| E01 | POST | `/api/products` | 167 | Không có middleware JWT |
| E02 | PUT | `/api/products/:id` | 179 | Không có middleware JWT |
| E03 | DELETE | `/api/products/:id` | 191 | Không có middleware JWT |
| E04 | POST | `/api/categories` | 249 | Chỉ authenticateToken |
| E05 | PUT | `/api/categories/:id` | 257 | Chỉ authenticateToken |
| E06 | DELETE | `/api/categories/:id` | 269 | Chỉ authenticateToken |
| E07 | POST | `/api/admin/import-products` | 199 | Chỉ authenticateToken |
| E08 | POST | `/api/admin/coupons` | 457 | Chỉ authenticateToken |
| E09 | DELETE | `/api/admin/coupons/:id` | 483 | Chỉ authenticateToken |
| E10 | GET | `/api/admin/users` | 494 | Chỉ authenticateToken |
| E11 | DELETE | `/api/admin/users/:id` | 504 | Chỉ authenticateToken |
| E12 | GET | `/api/admin/orders` | 510 | Chỉ authenticateToken |
| E13 | PUT | `/api/admin/orders/:id/status` | 525 | Chỉ authenticateToken |

Có 13 method/route thuộc phạm vi FR-12 đã triển khai. Không có POST/PUT/DELETE /api/coupons ở namespace public, không có PUT /api/admin/coupons/:id; không tạo endpoint giả để lấp danh sách. GET products/categories công khai không trở thành Admin-only. GET /api/coupons có JWT nhưng không ở /admin và không mutation; không tự kết luận phải Admin-only từ FR-12. apply-coupon/coupon-usage là workflow khách hàng, không tự đồng nhất với Admin coupon CRUD.

## Step 1 - Identify Input & Output Variables

| Mã | Input | Ý nghĩa | Ràng buộc SRS |
| --- | --- | --- | --- |
| I01 | JWT / Authorization | Credential gửi đến protected endpoint. | JWT hợp lệ là điều kiện cần; thiếu hoặc không hợp lệ không được thực hiện hành động bảo vệ. A12-02/A12-03. |
| I02 | role claim trong JWT đã verify | Quyền mà server phải kiểm tra sau khi JWT hợp lệ. | Chỉ admin; JWT User vẫn hợp lệ về xác thực nhưng không đủ quyền. Không dùng role từ body hoặc UI thay role token. |
| I03 | Method/route được bảo vệ | Điểm thực thi chính sách; mỗi route có middleware riêng trong source. | Cả 13 route ở inventory đều chịu cùng quy tắc A12-02/A12-03; route hợp lệ không có nghĩa actor được phép. |
| I04 | HTTP method | GET/POST/PUT/DELETE thực sự có route. | GET /admin cũng bảo vệ; mutation ngoài /admin vẫn bảo vệ. Không đặt oracle cho method không được SRS định nghĩa. |
| I05 | Body nghiệp vụ nominal | Dữ liệu giữ đúng để cô lập authorization. | Fixture dựa SRS FR-10/14/15/16/17/19; validation payload là phụ thuộc, không nhân Invalid EC nghiệp vụ vào báo cáo quyền. |
| I06 | Resource ID / trạng thái fixture | Đích tồn tại, đúng trạng thái khi mutation. | Giữ hợp lệ để không nhầm 404/invalid transition thành từ chối authorization. Không có Boundary Value ID từ FR-12. |
| I07 | Nguồn phiên Mobile | JWT có provenance lấy từ native Mobile login; sau đó replay bằng API tool nếu UI không gọi được route. | Giữ business rules FR-12; UI-only và API replay là hai tầng bằng chứng khác nhau. Không giả định Admin UI trên Mobile. |

| Mã | Output | Expected Result theo SRS |
| --- | --- | --- |
| O01 | Quyết định authorization | Chỉ JWT hợp lệ với role=admin vượt qua gate bảo vệ. Điều kiện quyền đúng không tự bảo đảm mọi business operation thành công. |
| O02 | Dữ liệu đọc được | Request thiếu/sai credential hoặc không phải Admin không được đọc dữ liệu protected Admin; không nhầm dữ liệu products công khai với rò rỉ Admin. |
| O03 | Side effect mutation | Request không đủ quyền không tạo/sửa/xóa dữ liệu. Phải snapshot/read-back, không chỉ nhìn thông báo lỗi. |
| O04 | Bằng chứng UI/API và provenance | Native UI chứng minh login/session có thật; API replay chứng minh shared backend authorization bằng token phiên Mobile. Không báo replay là bấm control Admin trên Mobile. |

Nếu chưa có cách capture JWT từ login bằng công cụ ngoài ứng dụng được cho phép, ghi chưa đủ Preconditions cho replay; không tự thêm màn hình hay sửa source, không thay API login trực tiếp rồi gọi đó là bằng chứng Mobile. Không yêu cầu token server bị revoke sau logout vì SRS chưa nêu.

## Các vấn đề cần human review

1. Xác nhận việc đánh giá Pool D chấp nhận native login + Backend API replay theo phiên Mobile khi không có native Admin UI. Không tự tạo FR-12 Mobile mới.
2. Cần controlled traffic capture để nối token với native login. Nếu không có công cụ/quyền hoặc không truy cập được IP hardcode, giữ ca replay ở trạng thái chưa đủ Preconditions, không sửa source.
3. Token claim role thiếu/khác admin thuộc lớp không đủ quyền; đại diện User là token thực có thể lấy. Không tự ký token thiếu claim bằng secret để giả lập issuer.
4. Không có chính sách token expiry/revocation sau logout trong SRS; không biến logout cục bộ thành yêu cầu token cũ vô hiệu trên server.
5. PUT category có route thật và phải bảo vệ theo FR-12, nhưng hành vi Update category không được FR-14 mô tả chi tiết. Ca ADM kiểm tra gate quyền; không tự thêm hợp đồng CRUD category.
6. Admin đạt điều kiện quyền không đồng nghĩa mọi business operation phải luôn thành công; đối chiếu fixture và SRS chức năng phụ thuộc. Ca âm phải xác minh không có đọc dữ liệu protected/side effect, không chỉ nhận lỗi.
7. Không dùng việc Mobile không có nút Admin làm bằng chứng backend đã an toàn; source chỉ là rủi ro tĩnh, chưa xác nhận bug.

Đã hoàn thành đến Step 1. Các Step sau chưa được đưa vào phiên bản commit này.
