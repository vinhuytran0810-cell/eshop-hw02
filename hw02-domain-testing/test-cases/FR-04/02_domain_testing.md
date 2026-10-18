# Domain Testing cho FR-04 - Quản lý hồ sơ cá nhân (Web)

| Thông tin | Nội dung |
| --- | --- |
| Ngày lập | 2026-10-18, múi giờ Asia/Saigon |
| Git revision đã đọc | `85af3ba875c88283615e22cb108f13e2fccaf0e9` |
| Repository hiện tại | `E:/KiemThu/eshop-sut` |
| Nguồn Expected Result | `README.md`, SRS phiên bản 2.0, cập nhật 2026-05-14 |
| Tài liệu tham khảo nội bộ | `hw02-domain-testing/reports/01_repository_analysis.md`; chỉ dùng để định hướng đọc source code |
| Phạm vi | Domain Testing cho cập nhật hồ sơ trên Web và Backend API phục vụ chức năng này |
| Trạng thái | Thiết kế để người học kiểm tra; chưa build, chạy ứng dụng hoặc thực thi Test Case; không có kết quả thực tế, Pass/Fail hay bug được xác nhận |

## Căn cứ SRS và phạm vi diễn giải

Các mã `P04-*` dưới đây giữ cách đánh mã của báo cáo repository; đây là mã truy vết của báo cáo, không phải yêu cầu mới.

| Mã yêu cầu | Nội dung SRS dùng làm Expected Result | Vị trí |
| --- | --- | --- |
| P04-01 | Người dùng đã đăng nhập có thể cập nhật Họ Tên, Số điện thoại và Địa chỉ giao hàng mặc định. | `README.md:64` |
| P04-02 | Số điện thoại hợp lệ bắt đầu bằng `0`, gồm 10-11 chữ số. | `README.md:65` |
| P04-03 | Email không được phép thay đổi qua giao diện. | `README.md:66` |
| P04-04 | Người dùng chỉ cập nhật được hồ sơ của chính mình. | `README.md:67` |
| P04-05 | Người dùng không thể tự thay đổi `role`. | `README.md:67`; SEC-06 tại dòng 283 củng cố cùng yêu cầu |
| AUTH-01 | Client lưu JWT và gửi qua `Authorization: Bearer <token>` trong yêu cầu có xác thực; API bảo mật yêu cầu JWT hợp lệ. | FR-02 trong `README.md:43`; SEC-02 tại dòng 279 là yêu cầu bảo mật tham khảo, đọc cùng điều kiện đăng nhập của P04-01 |

Phân tích này không chuyển các điều kiện đăng ký của FR-01 thành điều kiện cập nhật hồ sơ. Không đặt giới hạn tên/địa chỉ, quy tắc định dạng địa chỉ, tính duy nhất của số điện thoại hoặc quy tắc email mới. Các yêu cầu bố cục chung của FR-21/FR-22 là phạm vi GUI liên quan, không được dùng để tự bổ sung business rules cho dữ liệu FR-04. Lịch sử đơn hàng nằm cùng component nhưng thuộc FR-10/FR-11, không nằm trong thiết kế này.

### Source code đã đối chiếu

| Source code | Hàm/vị trí liên quan | Quan sát triển khai, không phải Expected Result |
| --- | --- | --- |
| `frontend-web/src/App.jsx:55` | `App()`, route `/profile` | Route Web dẫn tới `Profile`. |
| `frontend-web/src/pages/Profile.jsx:5-64,108-165` | `Profile()`, `handleUpdate()` | Form chứa `name`, `phone`, biến React `shippingAddress`; request gửi `shipping_address`. Email bị `disabled`; tên có `required`. Regex kiểm tra phone tại dòng 43 là `/^[1-9][0-9]{8,9}$/`; không dùng regex này làm yêu cầu đúng. |
| `frontend-web/src/context/AuthContext.jsx:6-40` | `AuthProvider()`, `login()`, `logout()` | Lưu token trong `localStorage`, gọi `GET /api/users/me` để nạp `user`, dùng Bearer header. |
| `backend/server.js:100-110` | `authenticateToken()` | Kiểm tra JWT và gắn `req.user`; mã HTTP triển khai không trở thành Expected Result của SRS. |
| `backend/server.js:112-135` | Callback của `GET /api/users/me`, `PUT /api/users/me` | PUT lấy `name`, `shipping_address`, `phone`, `role`; định danh dòng bằng `req.user.id`. Không thấy validation phone trong callback; có nhánh cập nhật `role`. Đây chỉ là dấu hiệu cần kiểm tra sau này. |
| `backend/database.js:50-61` | `initDatabase()`, bảng `users` | Lưu `id`, `name`, `email`, `role`, `shipping_address`, `phone`. Không nhập module để đọc dữ liệu vì `initDatabase()` được gọi tự động và có lệnh xóa/tạo lại bảng. |

Endpoint được xác minh: `POST /api/login`, `GET /api/users/me`, `PUT /api/users/me`. Không giả định tồn tại `PUT /api/users/:id`. Việc gửi thêm `id` trong một request bên dưới là dữ liệu đối kháng để thử yêu cầu tác động người khác, không phải khẳng định API công bố hỗ trợ trường này.

## Step 1 - Identify Input & Output Variables

### 1.1 Bảng Input Variables

| Mã biến | Input / ngữ cảnh | Ý nghĩa và điểm đi vào | Ràng buộc nghiệp vụ từ SRS | Giới hạn phân tích |
| --- | --- | --- | --- | --- |
| I-01 | Phiên đăng nhập / `token` | Định danh người thao tác; trên API được gửi trong `Authorization`. | Cập nhật yêu cầu người dùng đã đăng nhập và JWT hợp lệ: P04-01, AUTH-01. | Không có thời hạn JWT hay độ dài token do SRS quy định. Không dùng lỗi đăng nhập để thay cho kiểm thử quyền cập nhật. |
| I-02 | `name` | Họ Tên mới, qua form và body PUT. | Có thể cập nhật Họ Tên: P04-01. | Chưa có yêu cầu bắt buộc khi cập nhật, min/max, trim hay tập ký tự. Chỉ chọn giá trị tên thông thường; không suy ra Invalid EC từ `required` của HTML. |
| I-03 | `phone` | Số điện thoại mới; xử lý như chuỗi để giữ số `0` đầu. | Bắt đầu bằng `0`, gồm 10-11 chữ số: P04-02. | Phân lớp chuỗi không rỗng được cung cấp. Việc bỏ trường, `null`, chuỗi rỗng hay kiểu JSON khác cần làm rõ; chưa suy ra quy tắc bắt buộc. |
| I-04 | `shipping_address` | Địa chỉ giao hàng mặc định mới; trên React là `shippingAddress`. | Có thể cập nhật địa chỉ mặc định: P04-01. | Không có yêu cầu min/max, bắt buộc, định dạng hay chuẩn hóa. |
| I-05 | Thao tác với `email` trên Web UI | Giữ nguyên email hoặc cố sửa qua control gốc của trang `/profile`. | Không được thay đổi email qua UI: P04-03. | Đây là biến thao tác, không phải trường PUT hợp lệ. Không suy ra lệnh cấm thay email qua một API khác chưa được đặc tả. |
| I-06 | Đối tượng hồ sơ yêu cầu tác động | Quan hệ giữa người có token và chủ hồ sơ; không phải ô nhập có sẵn trên Web. | Chỉ hồ sơ của chính người thao tác được cập nhật: P04-04. | API hiện tại là `/me`; thử gửi `id` của tài khoản khác chỉ kiểm tra một kênh giả mạo mục tiêu đã chọn. |
| I-07 | Yêu cầu thay đổi `role` | Không yêu cầu đổi role hoặc chèn trường `role` khác giá trị hiện tại trong body API. | Không tự thay đổi `role`: P04-05. | Web không có control sửa role. Không tạo lớp role không hợp lệ theo enum vì FR-04 không quy định enum đầu vào cho role. |

### 1.2 Bảng Output Variables

| Mã biến | Output | Ý nghĩa cần quan sát khi thực thi sau này | Expected Result và căn cứ |
| --- | --- | --- | --- |
| O-01 | Kết quả cho phép cập nhật | Thao tác được áp dụng hay bị ngăn; phản hồi UI/API là một kênh quan sát. | Với dữ liệu hợp lệ và đúng quyền, phải cập nhật được (P04-01). Với phone không hợp lệ, không được lưu phone đó như phone hợp lệ (P04-02). Không ấn định mã HTTP, JSON schema, câu thông báo hoặc cơ chế alert. |
| O-02 | Hồ sơ sau cập nhật | `name`, `phone`, `shipping_address` hiển thị/đọc lại sau lưu và tải lại. | Giá trị hợp lệ đã gửi phải được lưu cho chính người thao tác, gồm nguyên số `0` đầu phone (P04-01/P04-02/P04-04). Một thông báo thành công đơn lẻ chưa chứng minh dữ liệu đã lưu. |
| O-03 | Email hiển thị và khả năng sửa | Email trước/sau thao tác UI; trạng thái có thể chỉnh sửa control. | Không thể thay đổi email qua UI gốc; email giữ nguyên trong thao tác này (P04-03). SRS không bắt buộc dùng đúng thuộc tính `disabled`. |
| O-04 | Phạm vi thay đổi hồ sơ | Hồ sơ nào bị thay đổi, đặc biệt hồ sơ người khác. | Không được ghi thay đổi lên hồ sơ người khác bằng phiên của người thao tác (P04-04). |
| O-05 | `role` sau thao tác | Role lưu trong hồ sơ, đối chiếu trước/sau. | Không được tự thay đổi role (P04-05). Không suy ra token cũ phải được thu hồi hoặc cập nhật tức thì. |

SRS chưa đủ cơ sở để xác định mọi output kỹ thuật hay một quy tắc rollback toàn bộ khi request có trường không hợp lệ. Với request giả mạo `id`/`role`, hệ thống có thể từ chối request hoặc bỏ qua trường bị cấm; bất biến bắt buộc là không sửa người khác và không đổi role. Với phone sai, chỉ khẳng định phone sai không được ghi; không tự yêu cầu tên/địa chỉ phải rollback toàn bộ.

## Step 2 - Identify Equivalence Classes

### 2.1 Nguyên tắc phân lớp

- Chỉ phân EC có cơ sở từ SRS; không ép mỗi Input phải có Invalid EC khi SRS không cung cấp điều kiện xác định tính không hợp lệ.
- Phân lớp `phone` theo thứ tự: độ dài chuỗi, thành phần chữ số, rồi số đầu. Thứ tự này làm các EC không chồng lấn. Representative Value của mỗi Invalid EC chỉ vi phạm một điều kiện phone, dù một chuỗi bất kỳ trong lớp có thể đồng thời có đặc điểm sai khác.
- Với phone có độ dài 10 hoặc 11, chuỗi có ký tự không phải chữ số vào P-05; chỉ chuỗi toàn chữ số mới xét tiếp số đầu ở P-06. Các dạng khác nhau như chữ cái/dấu cách không tách thành EC riêng nếu SRS không quy định cách xử lý khác nhau.
- Hai độ dài hợp lệ của phone được giữ thành hai Valid EC để truy vết rõ cả hai độ dài mà SRS cho phép; Expected Result nghiệp vụ giống nhau. Không tạo thêm tổ hợp tên x địa chỉ x phone.
- `email`, mục tiêu hồ sơ và `role` được phân theo thao tác/quyền, không theo các định dạng chuỗi mà FR-04 chưa quy định.

### 2.2 Bảng Equivalence Classes

| EC ID | Input | Điều kiện của EC | Loại | SRS | Lý do phân chia |
| --- | --- | --- | --- | --- | --- |
| FR04-EC-A-01 | I-01 | Có phiên đăng nhập và JWT hợp lệ của người thao tác. | Valid | P04-01, AUTH-01 | Điều kiện để cập nhật hồ sơ cá nhân. |
| FR04-EC-A-02 | I-01 | Không có phiên/JWT; API không có Authorization header. | Invalid | P04-01, AUTH-01 | Thiếu bằng chứng xác thực; khác với token được gửi nhưng không hợp lệ. |
| FR04-EC-A-03 | I-01 | Có token được gửi nhưng token không hợp lệ. | Invalid | P04-01, AUTH-01 | Có credential nhưng không chứng minh được danh tính. Chọn token sai chữ ký để không thêm quy tắc về thời hạn. |
| FR04-EC-N-01 | I-02 | Giá trị Họ Tên thông thường, có nội dung, được yêu cầu cập nhật. | Valid | P04-01 | Đại diện khả năng cập nhật tên. Chưa có cơ sở tạo Invalid EC theo rỗng/độ dài/ký tự. |
| FR04-EC-P-01 | I-03 | Chuỗi 10 chữ số, bắt đầu bằng `0`. | Valid | P04-02 | Bao phủ độ dài hợp lệ 10. |
| FR04-EC-P-02 | I-03 | Chuỗi 11 chữ số, bắt đầu bằng `0`. | Valid | P04-02 | Bao phủ độ dài hợp lệ 11, không bỏ sót nhánh hợp lệ còn lại. |
| FR04-EC-P-03 | I-03 | Chuỗi không rỗng có độ dài từ 1 đến 9 ký tự. | Invalid | P04-02 | Không đủ độ dài; Representative Value giữ số đầu và thành phần chữ số đúng. |
| FR04-EC-P-04 | I-03 | Chuỗi có độ dài từ 12 ký tự trở lên. | Invalid | P04-02 | Vượt độ dài; tách khỏi thiếu độ dài vì nằm ở phía khác của khoảng hợp lệ. |
| FR04-EC-P-05 | I-03 | Chuỗi dài 10 hoặc 11 ký tự, có ít nhất một ký tự không phải chữ số. | Invalid | P04-02 | Vi phạm yêu cầu chỉ gồm chữ số; độ dài đúng. Representative Value vẫn bắt đầu bằng `0`. |
| FR04-EC-P-06 | I-03 | Chuỗi toàn chữ số, dài 10 hoặc 11, số đầu khác `0`. | Invalid | P04-02 | Chỉ sai số đầu khi thành phần và độ dài đúng. |
| FR04-EC-S-01 | I-04 | Giá trị địa chỉ giao hàng thông thường, có nội dung, được yêu cầu cập nhật. | Valid | P04-01 | Đại diện khả năng cập nhật địa chỉ. Chưa có cơ sở tạo Invalid EC theo độ dài/định dạng. |
| FR04-EC-E-01 | I-05 | Giữ nguyên email trong thao tác cập nhật UI. | Valid | P04-03 | Hành vi được phép với email hiện tại. |
| FR04-EC-E-02 | I-05 | Cố đổi email thành giá trị khác qua control UI gốc. | Invalid | P04-03 | Hành vi bị cấm, kể cả email mới có định dạng thông thường. Có thể bị ngăn ngay tại UI. |
| FR04-EC-T-01 | I-06 | Cập nhật hồ sơ của chính người có token, không chèn mục tiêu khác. | Valid | P04-04 | Phạm vi ghi dữ liệu được phép. |
| FR04-EC-T-02 | I-06 | Dùng token của A nhưng yêu cầu tác động hồ sơ B khác A. | Invalid | P04-04 | Chỉ quan hệ chủ sở hữu không hợp lệ; token A vẫn hợp lệ. |
| FR04-EC-R-01 | I-07 | Không yêu cầu thay đổi role; bỏ trường `role` khỏi payload. | Valid | P04-05 | Trạng thái trung tính của cập nhật thông thường. |
| FR04-EC-R-02 | I-07 | Người thao tác yêu cầu role khác role đang có của chính mình. | Invalid | P04-05 | Chỉ thao tác đổi role bị cấm; không đồng thời giả mạo người khác. |

Phân tích có 17 EC: 8 Valid EC và 9 Invalid EC. Đây là các EC đã xác định trên phạm vi có Expected Result từ SRS, không phải tuyên bố đã phân loại mọi giá trị JSON hay mọi tình huống chưa được đặc tả.

### 2.3 Những trường hợp chưa đủ cơ sở phân Valid/Invalid EC

| Input / trường hợp | Điều SRS chưa xác định | Cách xử lý trong báo cáo |
| --- | --- | --- |
| `name`: rỗng, thiếu trường, `null`, chỉ dấu cách, tên rất dài | Bắt buộc khi cập nhật, giới hạn, chuẩn hóa và xử lý thiếu trường. | Không gắn Invalid EC, không lấy `required` trong UI làm yêu cầu SRS. |
| `shipping_address`: rỗng, thiếu, `null`, rất dài hoặc khác định dạng địa chỉ | Tính bắt buộc, giới hạn, định dạng và chuẩn hóa. | Không tạo Invalid EC hoặc Boundary Value số học. |
| `phone`: rỗng, bỏ trường, `null`, kiểu JSON không phải chuỗi | FR-04 mô tả phone hợp lệ nhưng chưa nêu cho phép xóa/bỏ qua phone khi cập nhật hay không. | Cần người học làm rõ trước khi bổ sung EC. P-03 áp dụng chuỗi không rỗng, không âm thầm bao gồm chuỗi rỗng. |
| `email` trong request API tự tạo | P04-03 chỉ cấm đổi qua UI; không nêu hợp đồng đổi email qua API. | Không suy ra một Test Case API cấm email chỉ từ P04-03. |
| Token hết hạn, role/token thay đổi sau đăng xuất | Thời hạn, thu hồi và đồng bộ token chưa quy định. | Không đặt Expected Result mới về thời gian/thu hồi. |
| Output kỹ thuật | Mã HTTP, nguyên văn thông báo, schema JSON, rollback toàn bộ. | Chỉ đối chiếu kết quả nghiệp vụ và bất biến tương ứng; không ép cơ chế triển khai cụ thể. |

Output trong Step 1 được dùng làm điều kiện kiểm tra cho từng Test Case. Không tạo Test Case cố tình sản sinh output sai rồi tính đó là coverage thành công; việc output đúng/sai chỉ có thể được xác định sau thực thi.

## Step 3 - Select Best Representatives & Design Test Cases

### 3.1 Dữ liệu và điều kiện chuẩn bị cho lần thực thi sau

Các ký hiệu dưới đây là dữ liệu dự kiến, chưa phải fixture đã tạo hoặc được xác minh trên ứng dụng đang chạy.

| Ký hiệu | Giá trị / điều kiện |
| --- | --- |
| A, B | Hai tài khoản kiểm thử tồn tại và khác nhau, đều có `role = 'user'`; ghi nhận ID thực tế thành `U_A`, `U_B`. Không hardcode ID chưa xác minh. |
| `T_A`, `T_B` | JWT hợp lệ lấy bằng `POST /api/login` cho A/B trong lần thực thi sau. Không tự ký token hợp lệ bằng secret lấy từ source. |
| `P_A0`, `P_B0` | Snapshot hồ sơ A/B trước từng Test Case, lấy từ `GET /api/users/me` bằng token của đúng chủ hồ sơ; ghi `id`, `name`, `phone`, `shipping_address`, `email`, `role`. |
| `N*` | `Nguyễn Minh An` - tên mới dự kiến; khi chuẩn bị fixture, bảo đảm khác tên trước đó để quan sát thay đổi. |
| `S*` | `25 Nguyễn Trãi, Phường Bến Thành, TP.HCM` - địa chỉ mới dự kiến; bảo đảm khác giá trị trước đó. |
| `P10` | `0912345678` - chuỗi 10 chữ số hợp lệ. |
| `P11` | `01234567890` - chuỗi 11 chữ số hợp lệ theo đúng SRS; không bổ sung quy tắc đầu số nhà mạng. |
| `B(P)` | Body JSON gồm `name = N*`, `phone = P`, `shipping_address = S*`; mặc định không có `role`, `id`, `email`. |
| `E*` | `profile.changed@example.com` - chỉ là giá trị định thử nhập qua UI; phải khác email A. |
| `T_bad` | Biến thể của `T_A` có cùng payload danh tính A nhưng chữ ký bị thay đổi; không thay payload. Token này được chọn để kiểm tra JWT không hợp lệ, không kiểm tra thời hạn. |

Chuẩn bị snapshot độc lập trước mỗi Test Case; không nối thành một chuỗi phụ thuộc kết quả ca trước. Không restart backend giữa thao tác và đọc lại vì source có cơ chế khởi tạo lại database. Nếu setup hoặc một bước UI bị chặn, ghi nhận điểm chặn khi thực thi sau này; không tự chuyển Test Case Web UI thành Backend API để hoàn tất ca.

### 3.2 Bảng Representative Values

| EC ID | Representative Value / thao tác | Lý do lựa chọn |
| --- | --- | --- |
| FR04-EC-A-01 | Phiên A và Bearer `T_A` hợp lệ. | Có danh tính rõ, dùng lại với hồ sơ A để kết hợp các Valid EC. |
| FR04-EC-A-02 | Bỏ hẳn Authorization header khi PUT. | Đại diện thiếu xác thực mà payload nghiệp vụ vẫn hợp lệ; không gộp với token hỏng. |
| FR04-EC-A-03 | Bearer `T_bad`. | Chỉ credential không hợp lệ; payload vẫn thuộc A và các trường hồ sơ đúng. |
| FR04-EC-N-01 | `N* = Nguyễn Minh An`. | Tên thông thường có dấu; không cần thêm một EC riêng chỉ vì tiếng Việt. |
| FR04-EC-P-01 | `P10 = 0912345678`. | Đúng số đầu, toàn chữ số, đủ 10 chữ số. |
| FR04-EC-P-02 | `P11 = 01234567890`. | Đúng 11 chữ số được SRS cho phép; tránh dùng suy luận về số điện thoại thực tế ngoài SRS. |
| FR04-EC-P-03 | `01234567` - 8 chữ số. | Chỉ sai độ dài ngắn; chọn trong lớp thay vì mặc định dùng giá trị liền kề của BVA. |
| FR04-EC-P-04 | `0123456789012` - 13 chữ số. | Chỉ sai độ dài dài; số đầu và thành phần vẫn đúng. |
| FR04-EC-P-05 | `091234567a` - 10 ký tự. | Chỉ một ký tự chữ thay cho chữ số; số đầu và độ dài giữ hợp lệ. |
| FR04-EC-P-06 | `1912345678` - 10 chữ số. | Chỉ số đầu khác `0`; không chọn số ngắn vốn có thể bị chặn bởi điều kiện khác. |
| FR04-EC-S-01 | `S* = 25 Nguyễn Trãi, Phường Bến Thành, TP.HCM`. | Địa chỉ có nội dung thông thường; không kiểm tra một chuẩn địa chỉ chưa được SRS quy định. |
| FR04-EC-E-01 | Không sửa email; giữ email trong `P_A0`. | Kết hợp với cập nhật ba trường được phép. |
| FR04-EC-E-02 | Thử thay email A bằng `E*` qua control UI gốc. | Chỉ hành vi sửa email bị cấm; giá trị mới không tạo thêm lỗi định dạng. |
| FR04-EC-T-01 | `PUT /api/users/me` bằng `T_A`, body `B(P10)` không có định danh ghi đè. | Cập nhật đúng chủ hồ sơ theo endpoint có thật. |
| FR04-EC-T-02 | `T_A`, body `B(P10)` thêm duy nhất `id = U_B`. | Thử một kênh giả mạo mục tiêu; không cần dựng endpoint không tồn tại hoặc thêm đồng thời nhiều định danh. |
| FR04-EC-R-01 | Không gửi `role`; giữ `role` trong `P_A0`. | Không yêu cầu thay role trong thao tác bình thường. |
| FR04-EC-R-02 | `T_A`, body `B(P10)` thêm duy nhất `role = 'admin'`, trong khi A đang là `user`. | Giá trị role có thật trong SRS nhưng hành vi tự đổi bị cấm; chỉ một điều kiện nghiệp vụ không hợp lệ. |

### 3.3 Quy ước mô tả Test Steps và coverage

- **Web UI**: thao tác bằng control gốc tại `/profile`; việc đọc lại bằng GET chỉ là bước quan sát bổ sung, không thay thế việc submit qua UI.
- **Backend API**: gọi trực tiếp endpoint đã được xác minh; không đi qua regex/HTML validation của Web.
- Bộ bước **W**: (1) đăng nhập A, chụp snapshot; (2) mở `/profile`; (3) điền `N*`, phone được chỉ định, `S*`, giữ email; (4) bấm Cập nhật; (5) ghi phản hồi, tải lại và đối chiếu hồ sơ A/B qua phiên của từng chủ. Lần thực thi sau chỉ đọc B nếu cần xác minh bất biến phạm vi.
- Bộ bước **K**: (1) chuẩn bị A/B và snapshot; (2) tạo body và header theo ca; (3) gửi `PUT /api/users/me`; (4) ghi request/response; (5) đọc lại A/B bằng phiên hợp lệ riêng, so sánh các trường liên quan. Không coi một success message là bằng chứng đã lưu.
- `V10` là viết tắt coverage `{FR04-EC-A-01, FR04-EC-N-01, FR04-EC-P-01, FR04-EC-S-01, FR04-EC-T-01, FR04-EC-R-01}`. Email E-01 chỉ cộng khi ca thao tác Web UI giữ email, không gán EC thao tác UI cho ca API.
- `V11` là `V10` thay P-01 bằng P-02. `V(P-x)` là `V10` thay P-01 bằng Invalid EC phone tương ứng. Các ký hiệu này chỉ rút gọn bảng, không phải EC mới.
- Với ca API xác thực sai, T-01 mô tả mục tiêu tự khai của request không có định danh người khác; không khẳng định token sai đã xác thực được A. Các Valid EC đi kèm chỉ là ngữ cảnh/dữ liệu chuẩn, không chứng minh đã đi qua các nhánh xử lý sau điểm chặn.

### 3.4 Bảng Domain Test Cases - Web UI

| Test Case ID | ECs Covered | Input | Test Steps | Expected Output |
| --- | --- | --- | --- | --- |
| FR04-DT-W01 | `V10` + FR04-EC-E-01 | Phiên A; `N*`, `P10`, `S*`; email giữ nguyên; không đổi role/mục tiêu. | Thực hiện W, dùng `P10`; đối chiếu cả ba giá trị sau tải lại. | Cho phép cập nhật tên, phone 10 chữ số và địa chỉ của A. Phone giữ số `0` đầu; email và role không đổi; B không bị sửa. Căn cứ P04-01 đến P04-05. |
| FR04-DT-W02 | `V11` + FR04-EC-E-01 | Như W01 nhưng phone `P11`. | Thực hiện W độc lập, dùng `P11`. | Cho phép cập nhật với phone 11 chữ số; dữ liệu A đọc lại khớp giá trị hợp lệ; email/role không đổi, B không bị sửa. Căn cứ P04-01 đến P04-05. |
| FR04-DT-W03 | `V(P-03)` + FR04-EC-E-01 | Phiên A; `N*`, `01234567`, `S*`; các input khác hợp lệ. | Thực hiện W; ghi rõ nếu UI ngăn request; kiểm tra phone sau thao tác. | Không lưu phone 8 chữ số như một phone hợp lệ. Không ấn định UI hay backend phải là nơi chặn; email/role và hồ sơ B vẫn được bảo vệ. Không suy ra rollback toàn bộ tên/địa chỉ. Căn cứ P04-02 đến P04-05. |
| FR04-DT-W04 | `V(P-04)` + FR04-EC-E-01 | Như W03 nhưng phone `0123456789012`. | Thực hiện W với chuỗi 13 chữ số. | Không lưu phone vượt 11 chữ số như phone hợp lệ; các bất biến email/role/phạm vi vẫn giữ. Căn cứ P04-02 đến P04-05. |
| FR04-DT-W05 | `V(P-05)` + FR04-EC-E-01 | Như W03 nhưng phone `091234567a`. | Thực hiện W với chuỗi đúng 10 ký tự, bắt đầu `0`, có chữ. | Không lưu chuỗi có ký tự không phải chữ số như phone hợp lệ; các bất biến còn lại giữ nguyên. Căn cứ P04-02 đến P04-05. |
| FR04-DT-W06 | `V(P-06)` + FR04-EC-E-01 | Như W03 nhưng phone `1912345678`. | Thực hiện W với 10 chữ số, chỉ sai số đầu. | Không lưu phone không bắt đầu bằng `0` như phone hợp lệ; các bất biến còn lại giữ nguyên. Căn cứ P04-02 đến P04-05. |
| FR04-DT-W07 | FR04-EC-E-02; các điều kiện `V10` giữ hợp lệ | Phiên A; định thử nhập `E*` vào email; các trường được phép chuẩn bị `N*`, `P10`, `S*`. | (1) Đăng nhập A, mở `/profile`; (2) kiểm tra và thử focus/gõ/dán vào control email gốc nếu thao tác được; (3) nếu UI ngăn sửa thì ghi nhận điểm ngăn, không bỏ `disabled` bằng DevTools; nếu nhập được thì quan sát email sau submit/tải lại với các trường khác hợp lệ; (4) đối chiếu email A. | UI gốc không cho phép thay đổi email A. Việc control ngăn chỉnh sửa là kết quả đúng dự kiến, không phải thiếu bước cần vượt qua. Không yêu cầu UI dùng đúng cơ chế `disabled`. Căn cứ P04-03. |

W07 không cần chờ một request chứa email sửa đổi nếu control đã ngăn thao tác: mục tiêu của ca là giới hạn tại UI. Nếu xuất hiện tình huống bất thường khi thực thi sau này, chỉ ghi bằng chứng; báo cáo thiết kế này chưa kết luận bug.

### 3.5 Bảng Domain Test Cases - Backend API

| Test Case ID | ECs Covered | Input | Test Steps | Expected Output |
| --- | --- | --- | --- | --- |
| FR04-DT-K01 | `V10` | Bearer `T_A`; body `B(P10)`. | Thực hiện K, dùng body chuẩn; đọc lại ba trường, role và danh tính. | Cập nhật được `N*`, `P10`, `S*` cho A; role không đổi, B không bị sửa. Không suy ra thêm quy tắc email API ngoài FR-04. Căn cứ P04-01/P04-02/P04-04/P04-05. |
| FR04-DT-K02 | `V11` | Bearer `T_A`; body `B(P11)`. | Thực hiện K độc lập, dùng phone 11 chữ số. | Cập nhật được phone 11 chữ số bắt đầu `0`, cùng tên/địa chỉ hợp lệ cho A; role và phạm vi giữ đúng. Căn cứ P04-01/P04-02/P04-04/P04-05. |
| FR04-DT-K03 | `V(P-03)` | Bearer `T_A`; `B('01234567')`. | Thực hiện K, chỉ thay phone; đọc lại phone. | Không ghi phone 8 chữ số như phone hợp lệ; role A và B được bảo vệ. Không ấn định rollback toàn bộ hoặc mã HTTP. Căn cứ P04-02/P04-04/P04-05. |
| FR04-DT-K04 | `V(P-04)` | Bearer `T_A`; `B('0123456789012')`. | Thực hiện K, chỉ thay phone; đọc lại phone. | Không ghi phone 13 chữ số như phone hợp lệ; các bất biến còn lại giữ nguyên. Căn cứ P04-02/P04-04/P04-05. |
| FR04-DT-K05 | `V(P-05)` | Bearer `T_A`; `B('091234567a')`. | Thực hiện K, chỉ thay phone; đọc lại phone. | Không ghi phone có chữ như phone hợp lệ; các bất biến còn lại giữ nguyên. Căn cứ P04-02/P04-04/P04-05. |
| FR04-DT-K06 | `V(P-06)` | Bearer `T_A`; `B('1912345678')`. | Thực hiện K, chỉ thay phone; đọc lại phone. | Không ghi phone có số đầu khác `0` như phone hợp lệ; các bất biến còn lại giữ nguyên. Căn cứ P04-02/P04-04/P04-05. |
| FR04-DT-K07 | FR04-EC-A-02; N-01, P-01, S-01, T-01, R-01 giữ làm ngữ cảnh chuẩn | Không Authorization header; body `B(P10)` không có định danh khác. | Thực hiện K, bỏ duy nhất header xác thực; dùng token hợp lệ riêng để đọc lại snapshot sau request. | Không cho phép request không đăng nhập thực hiện cập nhật hồ sơ; hồ sơ A/B không bị request này thay đổi. Không yêu cầu mã lỗi cụ thể hoặc tự chuyển hướng tới trang nào. Căn cứ P04-01, AUTH-01. |
| FR04-DT-K08 | FR04-EC-A-03; N-01, P-01, S-01, T-01, R-01 giữ làm ngữ cảnh chuẩn | Bearer `T_bad`; body `B(P10)`. | Thực hiện K; chỉ chữ ký token không hợp lệ; đọc lại bằng `T_A`/`T_B` hợp lệ. | Không cho phép credential không hợp lệ thực hiện cập nhật; hồ sơ A/B không bị request này thay đổi. Không ấn định 401/403 hay nội dung lỗi. Căn cứ P04-01, AUTH-01. |
| FR04-DT-K09 | FR04-EC-T-02; A-01, N-01, P-01, S-01, R-01 hợp lệ | Bearer `T_A`; `B(P10)` thêm `id = U_B`; không thêm role/email. | (1) Lấy snapshot A/B; (2) PUT `/api/users/me` bằng `T_A` với body đã nêu; (3) đọc lại B bằng `T_B`, đồng thời A bằng `T_A`; (4) xác định nơi dữ liệu bị tác động. | B không bị thay đổi bởi request của A. Có thể từ chối request hoặc bỏ qua `id` và chỉ cập nhật A; SRS không bắt buộc từ chối trường thừa. Không mô tả `/me` là API chọn người tùy ý. Căn cứ P04-04. |
| FR04-DT-K10 | FR04-EC-R-02; A-01, N-01, P-01, S-01, T-01 hợp lệ | Bearer `T_A`; `B(P10)` thêm `role = 'admin'`; A trước đó có `role = 'user'`. | (1) Xác minh role A qua snapshot; (2) PUT `/api/users/me` bằng token A; (3) GET `/api/users/me` bằng phiên A hợp lệ để đọc role đã lưu; (4) đối chiếu trước/sau. Không chỉ giải mã token cũ để kết luận role không đổi. | Role A vẫn là `user`; không được tự đổi thành `admin`. Có thể từ chối request hoặc bỏ qua role và xử lý trường hợp lệ; không yêu cầu rollback toàn bộ. Căn cứ P04-05. |

Có 17 Domain Test Cases: 7 Web UI và 10 Backend API. Các ca phone xuất hiện ở hai tầng có mục tiêu khác nhau: Web UI kiểm tra luồng người dùng; Backend API kiểm tra độc lập khi không đi qua validation UI. Không nhân thêm tổ hợp tên/địa chỉ, không tách chữ/dấu cách thành nhiều ca tương đương, không thêm GET-only Test Case nếu việc đọc lại đã là bước quan sát của ca cập nhật.

Các token/ID chỉ là ký hiệu chuẩn bị. Không có request nào trong bảng đã được gửi và không có Test Case nào được đánh dấu Pass/Fail.

### 3.6 EC Coverage Matrix

Mỗi ô liệt kê các Test Case được thiết kế để bao phủ EC đó; dấu `—` nghĩa là không áp dụng ở tầng này hoặc không thiết kế tại tầng này. Đây là coverage thiết kế, không phải coverage thực thi. Với Valid EC ở ca Invalid, chỉ ghi các ca có mục tiêu chính đủ rõ để ma trận gọn; tập `V10`/`V11` trong bảng ca vẫn chỉ ra các EC kết hợp bổ sung.

| EC ID | Web UI | Backend API | Cách chứng minh coverage khi thực thi sau |
| --- | --- | --- | --- |
| FR04-EC-A-01 | W01-W07 | K01-K06, K09-K10 | Phiên/token hợp lệ là điều kiện chuẩn của các ca. |
| FR04-EC-A-02 | — | K07 | Cố cập nhật khi không có credential. |
| FR04-EC-A-03 | — | K08 | Cố cập nhật bằng token sai chữ ký. |
| FR04-EC-N-01 | W01, W02 | K01, K02 | Quan sát tên mới sau cập nhật hợp lệ. |
| FR04-EC-P-01 | W01 | K01 | Cập nhật phone hợp lệ 10 chữ số. |
| FR04-EC-P-02 | W02 | K02 | Cập nhật phone hợp lệ 11 chữ số. |
| FR04-EC-P-03 | W03 | K03 | Chỉ chọn phone quá ngắn; còn lại hợp lệ. |
| FR04-EC-P-04 | W04 | K04 | Chỉ chọn phone quá dài; còn lại hợp lệ. |
| FR04-EC-P-05 | W05 | K05 | Chỉ chọn thành phần phone sai. |
| FR04-EC-P-06 | W06 | K06 | Chỉ chọn số đầu phone sai. |
| FR04-EC-S-01 | W01, W02 | K01, K02 | Quan sát địa chỉ mới sau cập nhật hợp lệ. |
| FR04-EC-E-01 | W01-W06 | — | Giữ email trong thao tác UI, đối chiếu trước/sau. |
| FR04-EC-E-02 | W07 | — | Thử thao tác bị cấm; ngăn tại UI vẫn bao phủ EC thao tác này. |
| FR04-EC-T-01 | W01, W02 | K01, K02 | Cập nhật đúng A; dùng B làm đối chứng phạm vi. |
| FR04-EC-T-02 | — | K09 | Chỉ giả mạo mục tiêu B; không đồng thời đổi role hoặc dùng token hỏng. |
| FR04-EC-R-01 | W01, W02 | K01, K02 | Không yêu cầu đổi role; đối chiếu role trước/sau. |
| FR04-EC-R-02 | — | K10 | Chỉ yêu cầu tự đổi role. |

Tiền tố đầy đủ của Wxx là `FR04-DT-Wxx`, của Kxx là `FR04-DT-Kxx`. Mọi EC trong bảng Step 2 có ít nhất một Test Case tương ứng: 17/17 EC được liên kết trong thiết kế. Con số này không bao gồm các trường hợp chưa được đặc tả ở mục 2.3 và không khẳng định đã kiểm thử toàn bộ bảo mật hồ sơ hay mọi cách giả mạo định danh.

### 3.7 Kiểm tra nguyên tắc cô lập Invalid EC

| Test Case | Invalid EC duy nhất được chủ đích kích hoạt | Điều kiện khác được giữ hợp lệ |
| --- | --- | --- |
| W03 / K03 | P-03: độ dài ngắn | Số đầu `0`, toàn chữ số; phiên A, tên, địa chỉ, mục tiêu của A; không đổi email/role. |
| W04 / K04 | P-04: độ dài dài | Số đầu `0`, toàn chữ số; các điều kiện còn lại như trên. |
| W05 / K05 | P-05: ký tự không phải chữ số | Độ dài 10, số đầu `0`; các điều kiện còn lại như trên. |
| W06 / K06 | P-06: số đầu khác `0` | Độ dài 10, toàn chữ số; các điều kiện còn lại như trên. |
| W07 | E-02: cố sửa email qua UI | Phiên A; các giá trị hồ sơ dự kiến hợp lệ, mục tiêu A, không đổi role. |
| K07 | A-02: thiếu credential | Body `B(P10)` đầy đủ; không mục tiêu khác hoặc role mới. |
| K08 | A-03: credential không hợp lệ | Body `B(P10)` đầy đủ; token giữ nguyên payload danh tính, chỉ chữ ký thay đổi. |
| K09 | T-02: yêu cầu tác động người khác | Token A hợp lệ, `B(P10)`, không đổi role. |
| K10 | R-02: tự đổi role | Token A hợp lệ, `B(P10)`, không định danh người khác. |

Không gộp token hỏng, phone sai, `id` người khác và role mới trong một request. Không dùng số phone đúng theo regex triển khai nhưng sai SRS làm giá trị nominal để vượt qua UI: nominal phải đúng SRS.

## Step 4 - Identify Boundary Values

### 4.1 Biên có căn cứ SRS

Biến có thứ tự ở đây là `L = số ký tự của phone`, với điều kiện giữ thành phần toàn chữ số và số đầu `0`. Đây là độ dài chuỗi, không phải trị số của số điện thoại. Bước liền kề là một ký tự/chữ số. Khoảng hợp lệ của L là từ 10 đến 11, bao gồm hai đầu.

| Biên | Giá trị biên hợp lệ | Giá trị liền kề phía dưới | Giá trị liền kề phía trên | Ví dụ phone tương ứng và ý nghĩa |
| --- | --- | --- | --- | --- |
| Biên dưới L = 10 | 10 chữ số | 9 chữ số, không hợp lệ | 11 chữ số, hợp lệ | L9: `012345678`; L10: `0123456789`; L11: `01234567890`. |
| Biên trên L = 11 | 11 chữ số | 10 chữ số, hợp lệ | 12 chữ số, không hợp lệ | L10: `0123456789`; L11: `01234567890`; L12: `012345678901`. |

Các ví dụ trong bảng này chỉ minh họa Boundary Values, không phải Test Case BVA. Vì khoảng hợp lệ chỉ có hai độ dài, giá trị liền kề trong khoảng của biên này đồng thời là biên hợp lệ còn lại; không tạo giá trị 10.5 hoặc giá trị nominal nguyên ở giữa. Nếu cần nominal cho biến khác trong bước sau, dùng dữ liệu chuẩn ở mục 3.1; chưa lập bộ 2-Point/3-Point BVA Test Cases trong nhiệm vụ này.

### 4.2 Các biến không có biên số học để suy ra

| Biến | Kết luận theo SRS |
| --- | --- |
| `name` | Chưa có min/max; không áp dụng giới hạn 255 của sản phẩm vào tên người dùng. |
| `shipping_address` | Chưa có min/max; không tự đặt giới hạn độ dài địa chỉ. |
| Số đầu và thành phần `phone` | Là điều kiện phân loại; không có khái niệm một số điện thoại tối thiểu/tối đa theo trị số. Không biến đầu số `0` thành khoảng giá trị số học để làm BVA. |
| `token` / phiên | Không có độ dài hay thời hạn SRS; không mượn giá trị từ implementation để đặt biên. |
| `email` / thao tác sửa, mục tiêu hồ sơ, yêu cầu `role` | Các quan hệ/quyền dạng phân loại, không có thứ tự số học phù hợp cho BVA. |
| Output | Không có thời gian đáp ứng, kích thước JSON hoặc giới hạn số học đầu ra được FR-04 quy định. |

## Các điểm cần người học kiểm tra trước bước tiếp theo

1. Xác nhận phạm vi EC hiện tại chỉ xét phone không rỗng được cung cấp; làm rõ chính sách để rỗng/bỏ trường và partial update trước khi bổ sung ca.
2. Xác nhận các lớp tên/địa chỉ thông thường là phạm vi có cơ sở hiện tại; không tự gắn Invalid cho chuỗi rỗng, dài hay ký tự đặc biệt khi SRS chưa quy định.
3. Xác nhận cách đọc lại và snapshot hai tài khoản khi thực thi; giữ riêng bằng chứng UI và API, không chỉ dựa thông báo hoặc token cũ.
4. K09 chỉ đại diện một kênh giả mạo `id` trên endpoint có thật; không tuyên bố coverage mọi kênh IDOR.
5. Những khác biệt source/SRS như regex phone và nhánh nhận `role` chỉ là quan sát tĩnh; không có kết luận bug hay kết quả thực thi trong báo cáo này.

Đã hoàn thành phần thiết kế Domain Testing cho FR-04 và danh sách Boundary Values. Dừng để người học kiểm tra; chưa thiết kế Test Case BVA riêng, chưa thực thi kiểm thử và chưa chuyển sang tính năng khác.


