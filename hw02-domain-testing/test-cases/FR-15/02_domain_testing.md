# Domain Testing cho FR-15 - Product Management CRUD (Web Admin)

Ngày lập: 2026-10-18 (Asia/Saigon). Repository: `E:/KiemThu/eshop-hw02`, origin `vinhuytran0810-cell/eshop-hw02`. Source được đối chiếu ở `8f1982bb8802bd6216e3c4e47da15c2e779fb6b4`. SRS là `README.md` phiên bản 2.0; tham khảo báo cáo repository và FR-04 để thống nhất cách trình bày. Chỉ thiết kế; chưa build, chạy ứng dụng/API/Test Case; không có Pass/Fail hay bug xác nhận.

| Mã yêu cầu | Nội dung Expected Result | SRS |
| --- | --- | --- |
| P15-01 | Admin có thể Create, Read, Update, Delete sản phẩm. | `README.md:193` |
| P15-02 | name bắt buộc, tối đa 255 ký tự. | `README.md:195` |
| P15-03 | price bắt buộc, là số dương (>0). | `README.md:196` |
| P15-04 | category bắt buộc và chọn từ danh sách có sẵn. | `README.md:197` |
| P15-05 | Update một sản phẩm chỉ thay đổi sản phẩm đó; các sản phẩm khác giữ nguyên. | `README.md:198` |
| P15-AUTH | Product mutations yêu cầu JWT hợp lệ và role=admin. Product reads công khai không bị biến thành Admin-only bởi FR-12. | `README.md:176-179`; FR-02 dòng 43 |

Source: `frontend-admin/src/App.jsx`, `fetchData()` dòng 41, `handleLogin()` dòng 61, `handleProductSubmit()` dòng 105, `deleteProduct()` dòng 133, tab products dòng 337, form dòng 483-566 và bảng/control dòng 567-608. Backend: callback `GET /api/products` dòng 141, GET detail dòng 159, POST dòng 167, PUT dòng 179, DELETE dòng 191, GET categories dòng 243; `authenticateToken()` dòng 100 trong `backend/server.js`. Schema: `backend/database.js:23-26,64-71`. Không có controller/ORM model riêng. Quan sát tĩnh: form chưa khai báo maxLength cho name; price chưa có required/min; nhánh Update đổi tên mọi phần tử trong state UI nhưng SQL lọc theo ID. Đây không phải Expected Result hoặc kết luận bug.

## Step 1 - Identify Input & Output Variables

| Mã | Input / ngữ cảnh | Ý nghĩa | Ràng buộc SRS |
| --- | --- | --- | --- |
| I01 | JWT | Credential của người thao tác; dùng token thực cấp từ login. | Mutations yêu cầu JWT hợp lệ. Không đặt thời hạn/độ dài. P15-AUTH. |
| I02 | role | Quyền của tài khoản/token. | Admin được CRUD; mutations yêu cầu admin. User không phải Input luôn sai với public reads. |
| I03 | Operation | Create, Read, Update, Delete. | Bốn thao tác P15-01; không dựng method chưa triển khai. |
| I04 | productId | Chọn một sản phẩm tồn tại cho Read detail, Update, Delete. | Không có min/max, quy tắc ID không tồn tại hoặc cấm xóa sản phẩm đã mua trong FR-15. |
| I05 | name | Tên sản phẩm trong POST/PUT và form. | Bắt buộc, tối đa 255 ký tự. Quy tắc whitespace/Unicode chưa rõ. |
| I06 | price | Giá trị giá trong POST/PUT và form. | Có giá trị, diễn giải được thành số dương. Không suy ra integer-only, bước giá hoặc tối đa từ schema INTEGER. |
| I07 | category_id | Liên kết danh mục chọn từ dữ liệu hiện có. | Bắt buộc và thuộc danh sách categories. ID danh mục không phải khoảng liên tục. |
| I08 | description | Nội dung mô tả gửi cùng form. | FR-15 không quy định required/giới hạn/định dạng; chỉ dùng giá trị thông thường. |
| I09 | imageUrl | Giá trị ảnh gửi cùng form. | Không có URL-validation/required/max do FR-15 nêu; không tự thêm Invalid EC. |
| I10 | Tập sản phẩm đối chứng | Ít nhất hai sản phẩm P và Q có giá trị khác nhau trước Update. | Là Preconditions quan sát tính độc lập P15-05, không phải input yêu cầu cập nhật hàng loạt. |

| Mã | Output | Expected Result |
| --- | --- | --- |
| O01 | Kết quả CRUD | Dữ liệu hợp lệ đúng quyền thực hiện được thao tác tương ứng. Không ấn định HTTP/JSON/thông báo. |
| O02 | Danh sách/chi tiết đã lưu | Create xuất hiện đúng dữ liệu; Read đọc được sản phẩm; Update lưu đúng sản phẩm đích; Delete làm sản phẩm bị xóa không còn trong danh sách. |
| O03 | Validation | Không lưu sản phẩm có trường vi phạm P15-02/P15-03/P15-04 như sản phẩm hợp lệ. Không tự yêu cầu rollback toàn bộ các trường khác trong Update sai. |
| O04 | Tính độc lập | Ngay sau Update và sau refresh/đọc lại, Q và các sản phẩm ngoài P không bị thay đổi; phân biệt UI state và persistence. |
| O05 | Authorization | Mutation không được phép không làm thay đổi dữ liệu; không dùng việc UI login ngăn để kết luận API đã bảo vệ. |

Endpoint có thật: GET/POST `/api/products`; GET/PUT/DELETE `/api/products/:id`; GET `/api/categories`. CSV import thuộc FR-16, không tính vào CRUD. Không suy ra yêu cầu xác nhận xóa hoặc product-name uniqueness.

## Step 2 - Identify Equivalence Classes

| EC ID | Input / quan hệ | Điều kiện | Loại | SRS | Lý do phân chia |
| --- | --- | --- | --- | --- | --- |
| FR15-EC-A01 | I01 | JWT hợp lệ. | Valid | P15-AUTH | Credential đúng làm nominal. |
| FR15-EC-A02 | I01 | Không có JWT khi mutation. | Invalid | P15-AUTH | Thiếu xác thực khác token hỏng. |
| FR15-EC-A03 | I01 | JWT không hợp lệ khi mutation. | Invalid | P15-AUTH | Một đại diện credential sai. |
| FR15-EC-R01 | I02 | Admin thao tác CRUD. | Valid | P15-01/P15-AUTH | Actor được phép. |
| FR15-EC-R02 | I02 | JWT User hợp lệ yêu cầu mutation. | Invalid | P15-AUTH | Chỉ sai quyền. |
| FR15-EC-O01 | I03 | Create. | Valid | P15-01 | Mỗi operation có tác động riêng. |
| FR15-EC-O02 | I03 | Read. | Valid | P15-01 | Mỗi operation có tác động riêng. |
| FR15-EC-O03 | I03 | Update. | Valid | P15-01 | Mỗi operation có tác động riêng. |
| FR15-EC-O04 | I03 | Delete. | Valid | P15-01 | Mỗi operation có tác động riêng. |
| FR15-EC-I01 | I04 | Sản phẩm đích tồn tại. | Valid | P15-01 | Dùng dữ liệu có thật. |
| FR15-EC-N01 | I05 | Tên thông thường không rỗng, dài không quá 255 ký tự. | Valid | P15-02 | Tên có nội dung trong giới hạn. |
| FR15-EC-N02 | I05 | Không cung cấp giá trị tên (bỏ trường hoặc chuỗi rỗng). | Invalid | P15-02 | Cùng vi phạm required; dùng một đại diện, không nhân ca thiếu/rỗng khi SRS không tách cách xử lý. |
| FR15-EC-N03 | I05 | Tên dài hơn 255 ký tự. | Invalid | P15-02 | Vi phạm max riêng với required. |
| FR15-EC-P01 | I06 | Có giá trị giá là số dương. | Valid | P15-03 | Miền giá hợp lệ. |
| FR15-EC-P02 | I06 | Không cung cấp giá trị price. | Invalid | P15-03 | Required khác số 0 được cung cấp. |
| FR15-EC-P03 | I06 | price bằng 0. | Invalid | P15-03 | Điểm bị loại bởi >0; tách khỏi âm để không bỏ lỗi so sánh >=0. |
| FR15-EC-P04 | I06 | price là số âm. | Invalid | P15-03 | Miền số âm khác thiếu giá hoặc 0. |
| FR15-EC-P05 | I06 | Giá trị được cung cấp không diễn giải được thành số. | Invalid | P15-03 | Vi phạm yêu cầu là số. |
| FR15-EC-C01 | I07 | Danh mục đã chọn tồn tại trong danh sách. | Valid | P15-04 | Miền hợp lệ theo membership, không theo trị số ID. |
| FR15-EC-C02 | I07 | Không chọn/cung cấp danh mục. | Invalid | P15-04 | Vi phạm required. |
| FR15-EC-C03 | I07 | Có category_id nhưng không thuộc danh sách có sẵn. | Invalid | P15-04 | Có giá trị nhưng sai membership. |
| FR15-EC-D01 | I08 | Mô tả thông thường được cung cấp. | Valid | P15-01 | Trường UI có thật, chưa có Invalid theo SRS. |
| FR15-EC-V01 | I09 | Giá trị imageUrl thông thường được cung cấp. | Valid | P15-01 | Trường UI có thật, không suy ra URL-validation. |

Có 23 EC. Không ép Invalid EC cho operation, ID tồn tại, description, imageUrl hoặc tập đối chứng khi SRS không có điều kiện phù hợp. ID không tồn tại, tên chỉ dấu cách, cách đếm Unicode, thiếu trường ở partial Update và numeric string cần human review. N02 gộp thiếu/rỗng theo cùng required, không tách ca trùng chỉ do JSON shape. Giá số 0, âm, thiếu và không phải số tách theo điều kiện nghiệp vụ. Các ca mutation gửi đủ body, ngoại trừ trường required đang được cô lập.

## Step 3 - Select Best Representatives & Design Test Cases

### Preconditions, dữ liệu và bước dùng chung

B* gồm `name = Bàn phím cơ HW02`, `price = 100000`, `category_id = C.id`, `description = Sản phẩm kiểm thử HW02`, `imageUrl = https://placehold.co/300x300/png?text=HW02`. Với Update, chọn giá trị hợp lệ mới khác snapshot P để quan sát được thay đổi. P và Q là hai sản phẩm tồn tại khác nhau, tên/giá khác nhau; C.id phải xác minh từ GET categories. C_missing là ID không thuộc danh sách tại thời điểm thử. Không có fixture nào đã tạo trong nhiệm vụ này.

**W-Create**: đăng nhập Admin, mở tab products, điền form Thêm B*, bấm Lưu, ghi response/UI, refresh và GET để đọc lại. **W-Read**: mở danh sách, đối chiếu P với snapshot và GET. **W-Update**: snapshot toàn bộ sản phẩm, bấm Sửa P, đổi dữ liệu, Lưu; đối chiếu UI ngay lập tức rồi refresh/GET để tách state hiển thị và persistence. **W-Delete**: snapshot, Xóa P qua control gốc, đọc lại danh sách.

**K-Create/Read/Update/Delete**: xác minh fixture/token; gửi lần lượt POST products, GET list + GET detail P, PUT products/P.id, DELETE products/P.id tương ứng; ghi request/response; GET list/detail bằng phiên quan sát hợp lệ để đối chiếu. Mỗi ca độc lập, không dựa sản phẩm còn lại từ ca trước. Không khởi động backend hoặc import database.js trong quá trình thiết kế.

UI không có cách chọn category ngoài danh sách hoặc nhập giá dạng chữ thì các ca đó chỉ nằm ở API; không sửa DOM/source để biến chúng thành ca UI. B* là giá trị nghiệp vụ nominal; SRS không quy định bắt buộc JSON number thay vì numeric string do form gửi lên.

### Representative Values

| EC ID | Representative Value | Lý do lựa chọn |
| --- | --- | --- |
| FR15-EC-A01 | T_Admin từ login. | Không tự ký token. |
| FR15-EC-A02 | Bỏ Authorization. | Payload tạo sản phẩm vẫn đúng. |
| FR15-EC-A03 | T_Admin sai chữ ký, giữ payload. | Không làm sai role hoặc payload nghiệp vụ. |
| FR15-EC-R01 | T_Admin, role=admin. | Chuẩn cho ca validation. |
| FR15-EC-R02 | T_User, role=user, POST hợp lệ. | Không gán lỗi cho User đọc public API. |
| FR15-EC-O01 | Create | Không gom CRUD thành một hành động không quan sát được. |
| FR15-EC-O02 | Read | Không gom CRUD thành một hành động không quan sát được. |
| FR15-EC-O03 | Update | Không gom CRUD thành một hành động không quan sát được. |
| FR15-EC-O04 | Delete | Không gom CRUD thành một hành động không quan sát được. |
| FR15-EC-I01 | P.id xác minh từ GET products. | Không hardcode ID hoặc tạo ID boundary. |
| FR15-EC-N01 | Bàn phím cơ HW02. | Giá trị nội lớp, không cần tạo tổ hợp nhiều tên. |
| FR15-EC-N02 | name = chuỗi rỗng. | Không dùng chỉ dấu cách vì trim chưa đặc tả. |
| FR15-EC-N03 | A lặp 260 lần. | ASCII giúp đếm rõ; không dùng 256 để biến Domain case thành bộ BVA riêng. |
| FR15-EC-P01 | 100000. | Nội miền, không suy ra đây là giá tối thiểu. |
| FR15-EC-P02 | API bỏ price; UI để trống nếu cần quan sát. | Các trường khác vẫn đầy đủ. |
| FR15-EC-P03 | 0. | Chỉ sai giá, không để trống. |
| FR15-EC-P04 | -50000. | Giá trị rõ trong lớp âm. |
| FR15-EC-P05 | khong-phai-so. | Không coi chuỗi số từ HTML là sai khi SRS chưa quy định kiểu JSON/coercion. |
| FR15-EC-C01 | C.id xác minh từ GET categories. | Danh mục hiện có, không hardcode 1. |
| FR15-EC-C02 | API bỏ category_id. | Không đồng thời gửi ID danh mục không tồn tại. |
| FR15-EC-C03 | C_missing xác minh không có trong GET categories. | Giữ kiểu biểu diễn ID giống ID hợp lệ, chỉ thay membership. |
| FR15-EC-D01 | Sản phẩm kiểm thử HW02. | Không thêm luật độ dài/HTML. |
| FR15-EC-V01 | https://placehold.co/300x300/png?text=HW02. | Chỉ giá trị nominal, không yêu cầu mạng ảnh hoạt động như business rule. |

### Domain Test Cases - Web Admin UI

| Test Case ID | Preconditions | Inputs | Test Steps | Expected Results | ECs Covered |
| --- | --- | --- | --- | --- | --- |
| FR15-DT-W01 | Admin đăng nhập; C tồn tại; dữ liệu mới chưa dùng. | Dữ liệu B*; chế độ Thêm sản phẩm mới. | Dùng W-Create; chỉ control gốc; ghi danh sách trước/sau và đọc lại. | Tạo được sản phẩm có dữ liệu B*, xuất hiện khi đọc lại. P15-01 đến P15-04. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-O01 |
| FR15-DT-W02 | Admin đăng nhập; P tồn tại; snapshot P/Q và danh sách. | Mở tab products, xác định P. | Dùng W-Read; chỉ control gốc; ghi danh sách trước/sau và đọc lại. | Hiển thị/đọc được P với dữ liệu tương ứng đã lưu; không phát sinh mutation. P15-01. | FR15-EC-A01, FR15-EC-R01, FR15-EC-O02, FR15-EC-I01 |
| FR15-DT-W03 | Admin đăng nhập; P tồn tại; snapshot P/Q và danh sách. | Dữ liệu B*; chọn Sửa đúng P. | Dùng W-Update; chỉ control gốc; ghi danh sách trước/sau và đọc lại. | Chỉ P cập nhật B*; Q và mọi sản phẩm khác giữ nguyên ngay trên UI và sau refresh/GET. P15-01 đến P15-05. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-W04 | Admin đăng nhập; P tồn tại; snapshot P/Q và danh sách. | Chọn Xóa trên P. | Dùng W-Delete; chỉ control gốc; ghi danh sách trước/sau và đọc lại. | Xóa được P; danh sách đọc lại không còn P. Không ép mã trả của GET detail hoặc dialog xác nhận. P15-01. | FR15-EC-A01, FR15-EC-R01, FR15-EC-O04, FR15-EC-I01 |
| FR15-DT-W05 | Admin đăng nhập; form Create; C hợp lệ. | B* nhưng chỉ name rỗng. | Điền form gốc, thử Lưu; ghi nhận browser/client/server chặn ở đâu; nếu không gửi request thì không giả lập request trong ca UI. | Không tạo sản phẩm với trường sai như sản phẩm hợp lệ; Expected Result theo P15-02/P15-03, không theo validation đang có. | FR15-EC-A01, FR15-EC-R01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-N02, FR15-EC-O01 |
| FR15-DT-W06 | Admin đăng nhập; form Create; C hợp lệ. | B* nhưng chỉ name là A lặp 260 lần. | Điền form gốc, thử Lưu; ghi nhận browser/client/server chặn ở đâu; nếu không gửi request thì không giả lập request trong ca UI. | Không tạo sản phẩm với trường sai như sản phẩm hợp lệ; Expected Result theo P15-02/P15-03, không theo validation đang có. | FR15-EC-A01, FR15-EC-R01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-N03, FR15-EC-O01 |
| FR15-DT-W07 | Admin đăng nhập; form Create; C hợp lệ. | B* nhưng chỉ price=0. | Điền form gốc, thử Lưu; ghi nhận browser/client/server chặn ở đâu; nếu không gửi request thì không giả lập request trong ca UI. | Không tạo sản phẩm với trường sai như sản phẩm hợp lệ; Expected Result theo P15-02/P15-03, không theo validation đang có. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P03, FR15-EC-O01 |

### Domain Test Cases - Backend API

| Test Case ID | Preconditions | Inputs | Test Steps | Expected Results | ECs Covered |
| --- | --- | --- | --- | --- | --- |
| FR15-DT-K01 | Admin hợp lệ; C/P/Q đã xác minh theo thao tác. | Theo bảng endpoint; Create/Update dùng B*. | Dùng K-Create; Read gồm GET list và GET detail P; thao tác khác đọc lại GET products. | Sản phẩm mới lưu đúng dữ liệu hợp lệ. P15-01 đến P15-04. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-O01 |
| FR15-DT-K02 | Admin hợp lệ; C/P/Q đã xác minh theo thao tác. | Theo bảng endpoint; Create/Update dùng B*. | Dùng K-Read; Read gồm GET list và GET detail P; thao tác khác đọc lại GET products. | Đọc được danh sách và P; không tự coi string/number price khác nhau là lỗi nếu giá trị nghiệp vụ đúng. P15-01. | FR15-EC-A01, FR15-EC-R01, FR15-EC-O02, FR15-EC-I01 |
| FR15-DT-K03 | Admin hợp lệ; C/P/Q đã xác minh theo thao tác. | Theo bảng endpoint; Create/Update dùng B*. | Dùng K-Update; Read gồm GET list và GET detail P; thao tác khác đọc lại GET products. | Chỉ P thay đổi; Q/các sản phẩm ngoài P giữ nguyên qua GET. P15-05. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-K04 | Admin hợp lệ; C/P/Q đã xác minh theo thao tác. | Theo bảng endpoint; Create/Update dùng B*. | Dùng K-Delete; Read gồm GET list và GET detail P; thao tác khác đọc lại GET products. | P không còn trong danh sách sau Delete. P15-01. | FR15-EC-A01, FR15-EC-R01, FR15-EC-O04, FR15-EC-I01 |
| FR15-DT-V01 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ name = chuỗi rỗng. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-02 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-N02, FR15-EC-O01 |
| FR15-DT-V02 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ name = A lặp 260 lần. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-02 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-N03, FR15-EC-O01 |
| FR15-DT-V03 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ bỏ price. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P02, FR15-EC-O01 |
| FR15-DT-V04 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ price = 0. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P03, FR15-EC-O01 |
| FR15-DT-V05 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ price = -50000. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P04, FR15-EC-O01 |
| FR15-DT-V06 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ price = khong-phai-so. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P05, FR15-EC-O01 |
| FR15-DT-V07 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ bỏ category_id. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-04 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-D01, FR15-EC-V01, FR15-EC-C02, FR15-EC-O01 |
| FR15-DT-V08 | Admin hợp lệ; C/P/Q có thật; tạo mới độc lập. | Create: B* nhưng chỉ category_id = C_missing. | Dùng K-Create; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-04 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-D01, FR15-EC-V01, FR15-EC-C03, FR15-EC-O01 |
| FR15-DT-V09 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ name = chuỗi rỗng. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-02 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-N02, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-V10 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ name = A lặp 260 lần. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-02 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-N03, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-V11 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ bỏ price. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P02, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-V12 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ price = 0. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P03, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-V13 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ price = -50000. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P04, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-V14 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ price = khong-phai-so. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-03 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-P05, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-V15 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ bỏ category_id. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-04 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-D01, FR15-EC-V01, FR15-EC-C02, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-V16 | Admin hợp lệ; C/P/Q có thật; snapshot P trước Update. | Update: B* nhưng chỉ category_id = C_missing. | Dùng K-Update; giữ các trường còn lại nominal; đọc lại và đối chiếu trường sai. | Không lưu sản phẩm/giá trị vi phạm P15-04 như dữ liệu hợp lệ; Update không làm biến đổi sản phẩm khác (P15-05). Không ép rollback toàn bộ field hợp lệ hoặc mã HTTP. | FR15-EC-A01, FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-D01, FR15-EC-V01, FR15-EC-C03, FR15-EC-O03, FR15-EC-I01 |
| FR15-DT-A01 | C hợp lệ; body Create hoàn toàn hợp lệ; snapshot trước thao tác. | POST products B*; bỏ JWT. | Dùng K-Create; chỉ thay credential hoặc role, không thay cả hai; đọc lại bằng phiên Admin hợp lệ. | Mutation không được phép không tạo sản phẩm/đổi dữ liệu. P15-AUTH; không ép mã lỗi. | FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-A02, FR15-EC-O01 |
| FR15-DT-A02 | C hợp lệ; body Create hoàn toàn hợp lệ; snapshot trước thao tác. | POST products B*; token Admin sai chữ ký. | Dùng K-Create; chỉ thay credential hoặc role, không thay cả hai; đọc lại bằng phiên Admin hợp lệ. | Mutation không được phép không tạo sản phẩm/đổi dữ liệu. P15-AUTH; không ép mã lỗi. | FR15-EC-R01, FR15-EC-N01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-A03, FR15-EC-O01 |
| FR15-DT-A03 | C hợp lệ; body Create hoàn toàn hợp lệ; snapshot trước thao tác. | POST products B*; JWT User hợp lệ. | Dùng K-Create; chỉ thay credential hoặc role, không thay cả hai; đọc lại bằng phiên Admin hợp lệ. | Mutation không được phép không tạo sản phẩm/đổi dữ liệu. P15-AUTH; không ép mã lỗi. | FR15-EC-A01, FR15-EC-N01, FR15-EC-P01, FR15-EC-C01, FR15-EC-D01, FR15-EC-V01, FR15-EC-R02, FR15-EC-O01 |

### EC Coverage Matrix

| EC ID | Test Cases được thiết kế | Phạm vi |
| --- | --- | --- |
| FR15-EC-A01 | FR15-DT-W01, FR15-DT-W02, FR15-DT-W03, FR15-DT-W04, FR15-DT-K01, FR15-DT-K02, FR15-DT-K03, FR15-DT-K04, FR15-DT-V01, FR15-DT-V02, FR15-DT-V03, FR15-DT-V04, FR15-DT-V05, FR15-DT-V06, FR15-DT-V07, FR15-DT-V08, FR15-DT-V09, FR15-DT-V10, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-V15, FR15-DT-V16, FR15-DT-W05, FR15-DT-W06, FR15-DT-W07, FR15-DT-A03 | Coverage thiết kế; chưa thực thi |
| FR15-EC-A02 | FR15-DT-A01 | Coverage thiết kế; chưa thực thi |
| FR15-EC-A03 | FR15-DT-A02 | Coverage thiết kế; chưa thực thi |
| FR15-EC-R01 | FR15-DT-W01, FR15-DT-W02, FR15-DT-W03, FR15-DT-W04, FR15-DT-K01, FR15-DT-K02, FR15-DT-K03, FR15-DT-K04, FR15-DT-V01, FR15-DT-V02, FR15-DT-V03, FR15-DT-V04, FR15-DT-V05, FR15-DT-V06, FR15-DT-V07, FR15-DT-V08, FR15-DT-V09, FR15-DT-V10, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-V15, FR15-DT-V16, FR15-DT-W05, FR15-DT-W06, FR15-DT-W07, FR15-DT-A01, FR15-DT-A02 | Coverage thiết kế; chưa thực thi |
| FR15-EC-R02 | FR15-DT-A03 | Coverage thiết kế; chưa thực thi |
| FR15-EC-O01 | FR15-DT-W01, FR15-DT-K01, FR15-DT-V01, FR15-DT-V02, FR15-DT-V03, FR15-DT-V04, FR15-DT-V05, FR15-DT-V06, FR15-DT-V07, FR15-DT-V08, FR15-DT-W05, FR15-DT-W06, FR15-DT-W07, FR15-DT-A01, FR15-DT-A02, FR15-DT-A03 | Coverage thiết kế; chưa thực thi |
| FR15-EC-O02 | FR15-DT-W02, FR15-DT-K02 | Coverage thiết kế; chưa thực thi |
| FR15-EC-O03 | FR15-DT-W03, FR15-DT-K03, FR15-DT-V09, FR15-DT-V10, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-V15, FR15-DT-V16 | Coverage thiết kế; chưa thực thi |
| FR15-EC-O04 | FR15-DT-W04, FR15-DT-K04 | Coverage thiết kế; chưa thực thi |
| FR15-EC-I01 | FR15-DT-W02, FR15-DT-W03, FR15-DT-W04, FR15-DT-K02, FR15-DT-K03, FR15-DT-K04, FR15-DT-V09, FR15-DT-V10, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-V15, FR15-DT-V16 | Coverage thiết kế; chưa thực thi |
| FR15-EC-N01 | FR15-DT-W01, FR15-DT-W03, FR15-DT-K01, FR15-DT-K03, FR15-DT-V03, FR15-DT-V04, FR15-DT-V05, FR15-DT-V06, FR15-DT-V07, FR15-DT-V08, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-V15, FR15-DT-V16, FR15-DT-W07, FR15-DT-A01, FR15-DT-A02, FR15-DT-A03 | Coverage thiết kế; chưa thực thi |
| FR15-EC-N02 | FR15-DT-V01, FR15-DT-V09, FR15-DT-W05 | Coverage thiết kế; chưa thực thi |
| FR15-EC-N03 | FR15-DT-V02, FR15-DT-V10, FR15-DT-W06 | Coverage thiết kế; chưa thực thi |
| FR15-EC-P01 | FR15-DT-W01, FR15-DT-W03, FR15-DT-K01, FR15-DT-K03, FR15-DT-V01, FR15-DT-V02, FR15-DT-V07, FR15-DT-V08, FR15-DT-V09, FR15-DT-V10, FR15-DT-V15, FR15-DT-V16, FR15-DT-W05, FR15-DT-W06, FR15-DT-A01, FR15-DT-A02, FR15-DT-A03 | Coverage thiết kế; chưa thực thi |
| FR15-EC-P02 | FR15-DT-V03, FR15-DT-V11 | Coverage thiết kế; chưa thực thi |
| FR15-EC-P03 | FR15-DT-V04, FR15-DT-V12, FR15-DT-W07 | Coverage thiết kế; chưa thực thi |
| FR15-EC-P04 | FR15-DT-V05, FR15-DT-V13 | Coverage thiết kế; chưa thực thi |
| FR15-EC-P05 | FR15-DT-V06, FR15-DT-V14 | Coverage thiết kế; chưa thực thi |
| FR15-EC-C01 | FR15-DT-W01, FR15-DT-W03, FR15-DT-K01, FR15-DT-K03, FR15-DT-V01, FR15-DT-V02, FR15-DT-V03, FR15-DT-V04, FR15-DT-V05, FR15-DT-V06, FR15-DT-V09, FR15-DT-V10, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-W05, FR15-DT-W06, FR15-DT-W07, FR15-DT-A01, FR15-DT-A02, FR15-DT-A03 | Coverage thiết kế; chưa thực thi |
| FR15-EC-C02 | FR15-DT-V07, FR15-DT-V15 | Coverage thiết kế; chưa thực thi |
| FR15-EC-C03 | FR15-DT-V08, FR15-DT-V16 | Coverage thiết kế; chưa thực thi |
| FR15-EC-D01 | FR15-DT-W01, FR15-DT-W03, FR15-DT-K01, FR15-DT-K03, FR15-DT-V01, FR15-DT-V02, FR15-DT-V03, FR15-DT-V04, FR15-DT-V05, FR15-DT-V06, FR15-DT-V07, FR15-DT-V08, FR15-DT-V09, FR15-DT-V10, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-V15, FR15-DT-V16, FR15-DT-W05, FR15-DT-W06, FR15-DT-W07, FR15-DT-A01, FR15-DT-A02, FR15-DT-A03 | Coverage thiết kế; chưa thực thi |
| FR15-EC-V01 | FR15-DT-W01, FR15-DT-W03, FR15-DT-K01, FR15-DT-K03, FR15-DT-V01, FR15-DT-V02, FR15-DT-V03, FR15-DT-V04, FR15-DT-V05, FR15-DT-V06, FR15-DT-V07, FR15-DT-V08, FR15-DT-V09, FR15-DT-V10, FR15-DT-V11, FR15-DT-V12, FR15-DT-V13, FR15-DT-V14, FR15-DT-V15, FR15-DT-V16, FR15-DT-W05, FR15-DT-W06, FR15-DT-W07, FR15-DT-A01, FR15-DT-A02, FR15-DT-A03 | Coverage thiết kế; chưa thực thi |

Thiết kế 30 ca; không thêm mọi kết hợp name/price/category. Mỗi ca Vxx chỉ thay một Invalid EC, vẫn có Admin/JWT đúng và hai trường còn lại đúng. Áp dụng validation cho cả hai callback Create/Update độc lập; Read/Delete không có payload name/price/category để gán EC giả. Bốn ca UI và bốn ca API CRUD có mục tiêu tầng khác nhau; ba ca UI Invalid tập trung control/validation gốc. Ba ca quyền chỉ dùng POST đại diện, coverage mọi mutation route thuộc báo cáo FR-12. EC Coverage Matrix là thiết kế, không phải tỷ lệ đã chạy.

### Ma trận CRUD và validation

| Operation | Web Admin UI | Backend API | Trọng tâm |
| --- | --- | --- | --- |
| Create | FR15-DT-W01, W05-W07 | FR15-DT-K01, V01-V08, A01-A03 | Tạo mới, validation độc lập và quyền. |
| Read | FR15-DT-W02 | FR15-DT-K02 | Danh sách và chi tiết sản phẩm có thật; không mutation. |
| Update | FR15-DT-W03 | FR15-DT-K03, V09-V16 | Validation riêng callback PUT; UI trước refresh và persistence; đối chứng sản phẩm khác. |
| Delete | FR15-DT-W04 | FR15-DT-K04 | Sản phẩm bị xóa vắng khỏi danh sách sau đọc lại. |

## Các vấn đề cần human review

1. Làm rõ whitespace/trim và cách đếm ký tự Unicode của name; bộ biên minh họa dùng ASCII để tránh tự quyết định.
2. Làm rõ precision/kiểu biểu diễn giá và giá dương nhỏ nhất nếu muốn có giá trị liền kề số học. SRS không có max price.
3. Chưa có hợp đồng partial Update, ID không tồn tại, xóa sản phẩm đã có order hoặc rollback mọi field; không đặt Expected Result mới cho chúng.
4. UI selector không hỗ trợ category ngoài danh sách; chỉ API kiểm tra membership bất hợp lệ. Cần snapshot danh sách tại thời điểm thực thi để không nhầm concurrency.
5. Dấu hiệu đổi tên hàng loạt ở state UI chưa chứng minh SQL update nhiều dòng; ca Update bắt buộc tách hai tầng.
6. FR-15 không làm GET products thành Admin-only; các ca hạn chế quyền ở đây chỉ áp mutation.

Đã hoàn thành đến Step 3. Các Step sau chưa được đưa vào phiên bản commit này.
