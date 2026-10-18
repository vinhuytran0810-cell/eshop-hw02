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

## Các vấn đề cần human review

1. Làm rõ whitespace/trim và cách đếm ký tự Unicode của name; bộ biên minh họa dùng ASCII để tránh tự quyết định.
2. Làm rõ precision/kiểu biểu diễn giá và giá dương nhỏ nhất nếu muốn có giá trị liền kề số học. SRS không có max price.
3. Chưa có hợp đồng partial Update, ID không tồn tại, xóa sản phẩm đã có order hoặc rollback mọi field; không đặt Expected Result mới cho chúng.
4. UI selector không hỗ trợ category ngoài danh sách; chỉ API kiểm tra membership bất hợp lệ. Cần snapshot danh sách tại thời điểm thực thi để không nhầm concurrency.
5. Dấu hiệu đổi tên hàng loạt ở state UI chưa chứng minh SQL update nhiều dòng; ca Update bắt buộc tách hai tầng.
6. FR-15 không làm GET products thành Admin-only; các ca hạn chế quyền ở đây chỉ áp mutation.

Đã hoàn thành đến Step 2. Các Step sau chưa được đưa vào phiên bản commit này.
