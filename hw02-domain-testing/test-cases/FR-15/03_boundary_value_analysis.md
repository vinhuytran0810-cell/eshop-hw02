# Task 03 - Boundary Value Analysis: FR-15 - Product CRUD (Web Admin)

Ngày lập: 2026-10-18 (Asia/Saigon). Repository: `E:/KiemThu/eshop-hw02`, origin `vinhuytran0810-cell/eshop-hw02`. Revision nền: `6ef7f1ee88a20ccc1b38c90e7ce12ba768f9df2d`. SRS: `README.md`, phiên bản 2.0. Đã đối chiếu source và cả bốn báo cáo `02_domain_testing.md`; báo cáo liên quan trực tiếp ở cùng thư mục với file này.

Phương pháp theo Task 03: 2-point chọn (L−1,L) và (U,U+1) ở biên nguyên, đúng một Valid tại biên và một Invalid ngay ngoài; 3-point chọn (B−1,B,B+1), phân Valid/Invalid theo SRS và loại trùng giá trị. Không áp công thức ±1 cho miền chưa có bước nhảy. Chỉ thay một Input Variable; các Input khác nominal, hợp lệ. Source chỉ xác định đường thao tác, không định nghĩa Expected Result. Các báo cáo Domain Testing được giữ nguyên. Chưa build/chạy ứng dụng, gửi API hoặc thực thi Test Case; không có Pass/Fail hoặc bug xác nhận.

SRS `README.md:193-198`: Admin CRUD; name bắt buộc, tối đa 255 ký tự (P15-02); price bắt buộc và >0 (P15-03); category chọn từ danh sách hiện có (P15-04); Update chỉ thay sản phẩm đích (P15-05). FR-12 dòng 176-179 cho JWT/role mutations. Source: `frontend-admin/src/App.jsx`, handleProductSubmit() dòng 105-131, form 483-543, bảng 567-608; `backend/server.js:141-196,243-247`, callbacks products/categories; schema `backend/database.js:64-71`. HTML type=number, kiểu INTEGER và validation hiện tại không tạo quy tắc integer-only/step giá mới.

## 1. Khả năng áp dụng BVA cho từng Input Variable

| Input | Khả năng áp dụng | Căn cứ / giải thích |
| --- | --- | --- |
| JWT | N/A | Credential phân loại; không có độ dài/thời hạn đặc tả. |
| role | N/A | Quyền phân loại; giữ Admin hợp lệ. |
| Operation | N/A cho bản thân operation | Create/Update là hai ngữ cảnh kiểm tra cùng biên name; Read/Delete không có input tên mới. |
| productId | N/A | Định danh có membership; không có min/max nghiệp vụ. |
| name | Áp dụng theo độ dài L | Required diễn giải là chuỗi không rỗng có nội dung; chọn ASCII A lặp L lần, 1≤L≤255. |
| price | Nhận diện được điểm cắt; chưa đủ bộ BVA chuẩn | 0 Invalid, >0 Valid nhưng không có precision/step; không xác định duy nhất điểm liền kề. |
| category_id | N/A | Membership trong danh sách có sẵn, không phải khoảng ID. |
| description | N/A | Không có min/max trong FR-15. |
| imageUrl | N/A | Không có giới hạn chuỗi/URL từ FR-15. |
| Tập sản phẩm đối chứng | N/A | Preconditions để quan sát tính độc lập, không phải miền số để kiểm biên. |

## 2. Boundary Identification Table

| Boundary ID | Biến | Loại biên | Giá trị B | Tính hợp lệ / bước liền kề | SRS |
| --- | --- | --- | --- | --- | --- |
| FR15-B-L | L = độ dài name | Dưới | 1 | Inclusive, Valid tại biên; bước độ dài một ký tự ASCII. | P15-02, README.md:195 |
| FR15-B-U | L = độ dài name | Trên | 255 | Inclusive, Valid tại biên; bước độ dài một ký tự ASCII. | P15-02, README.md:195 |
| FR15-B-P0 | price | Điểm cắt bị loại | 0 | 0 Invalid; >0 Valid; chưa xác định step/giá liền kề. Bộ BVA chuẩn chưa đủ cơ sở. | P15-03, README.md:196 |

## 3. Bảng 2-point BVA và 3-point BVA

### 2-point BVA

| Boundary ID | 2-point BVA | Phân loại | Dữ liệu / giới hạn |
| --- | --- | --- | --- |
| FR15-B-L | (0, 1) | 0: Invalid; 1: Valid | L=0: chuỗi rỗng; L=1: A lặp 1 lần |
| FR15-B-U | (255, 256) | 255: Valid; 256: Invalid | L=255: A lặp 255 lần; L=256: A lặp 256 lần |
| FR15-B-P0 | Chưa xác định duy nhất | 0 Invalid; >0 Valid | Không có step nên không lập bộ sát biên chuẩn. |

### 3-point BVA

| Boundary ID | 3-point BVA | Phân loại | Dữ liệu / giới hạn |
| --- | --- | --- | --- |
| FR15-B-L | (0, 1, 2) | 0: Invalid; 1: Valid; 2: Valid | L=0: chuỗi rỗng; L=1: A lặp 1 lần; L=2: A lặp 2 lần |
| FR15-B-U | (254, 255, 256) | 254: Valid; 255: Valid; 256: Invalid | L=254: A lặp 254 lần; L=255: A lặp 255 lần; L=256: A lặp 256 lần |
| FR15-B-P0 | Chưa xác định duy nhất | 0 Invalid; >0 Valid | Không có step nên không lập bộ sát biên chuẩn. |

2-point sau loại trùng: {0,1,255,256}; 3-point sau loại trùng: {0,1,2,254,255,256}. Dùng chung Test Case cho giá trị thuộc cả hai phương pháp. Có 6 giá trị × 2 operation Create/Update × 2 tầng UI/API = 24 Test Cases vật lý. Đây là các context khác nhau cần phân biệt vì form/validation và hai backend callback khác nhau; không nhân các giá/category khác. Bộ 2-point dùng 16 trong 24 ca; bộ 3-point dùng cả 24, không cộng thành 40 ca.

Với price, không lập bộ 2-point/3-point chuẩn vì thiếu step/precision. 0 có oracle Invalid nhưng không xác định được duy nhất Valid sát bên hoặc bộ ba sát biên. Không tự dùng −1/0/1 hay −0.01/0/0.01; không thêm Test Case minh họa như chuẩn BVA. Domain Testing đã có ca price=0 (FR15-DT-V04, FR15-DT-V12, FR15-DT-W07); chỉ truy vết, không đếm thành ca BVA mới. Không đưa ví dụ giá dương tùy ý vào dataset chuẩn.

## 4. BVA Test Cases

Giữ Admin/T_Admin hợp lệ; C.id là danh mục đang tồn tại đã xác minh bằng GET /api/categories; price=100000; description=Mô tả sản phẩm HW02; imageUrl=https://placehold.co/300x300/png?text=HW02. B(L) gồm name=A lặp L lần và các trường nominal vừa nêu. L=0 nghĩa là gửi chuỗi rỗng, không phải số 0, null hoặc bỏ field.

Create dùng fixture mới độc lập; Update dùng P tồn tại và ít nhất một sản phẩm Q đối chứng. Trước Update chụp P và danh sách sản phẩm; chỉ thay name, các trường khác lấy nominal đã xác lập giống nhau cho mọi ca. Quan hệ ID/trạng thái/role luôn hợp lệ; dữ liệu đối chứng dùng xác minh P15-05, không thay đổi thành một input không hợp lệ khác. Không có fixture nào được tạo trong nhiệm vụ này.

### Web Admin UI / Create

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| FR15-BVA-WC-N0 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(0), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) mở form Thêm; (3) điền/nhập thử name=A lặp 0 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Không tạo sản phẩm có name rỗng như tên hợp lệ. P15-02. Nếu UI ngăn nhập/lưu, ghi điểm chặn và không tự coi backend đã được kiểm tra. | FR15-B-L/2-point/dưới; FR15-B-L/3-point/dưới; EC Domain: FR15-EC-N02 |
| FR15-BVA-WC-N1 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(1), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) mở form Thêm; (3) điền/nhập thử name=A lặp 1 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Tạo được sản phẩm có name đúng A lặp 1 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/2-point/tại biên; FR15-B-L/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-WC-N2 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(2), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) mở form Thêm; (3) điền/nhập thử name=A lặp 2 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Tạo được sản phẩm có name đúng A lặp 2 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/3-point/trên; EC Domain: FR15-EC-N01 |
| FR15-BVA-WC-N254 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(254), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) mở form Thêm; (3) điền/nhập thử name=A lặp 254 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Tạo được sản phẩm có name đúng A lặp 254 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/3-point/dưới; EC Domain: FR15-EC-N01 |
| FR15-BVA-WC-N255 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(255), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) mở form Thêm; (3) điền/nhập thử name=A lặp 255 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Tạo được sản phẩm có name đúng A lặp 255 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/2-point/tại biên; FR15-B-U/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-WC-N256 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(256), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) mở form Thêm; (3) điền/nhập thử name=A lặp 256 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Không tạo sản phẩm có name dài 256 ký tự như tên hợp lệ. P15-02. Nếu UI ngăn nhập/lưu, ghi điểm chặn và không tự coi backend đã được kiểm tra. | FR15-B-U/2-point/trên; FR15-B-U/3-point/trên; EC Domain: FR15-EC-N03 |

### Web Admin UI / Update

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| FR15-BVA-WU-N0 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(0), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) bấm Sửa đúng P; (3) điền/nhập thử name=A lặp 0 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Không ghi đè name của P bằng name rỗng như tên hợp lệ. Các sản phẩm ngoài P vẫn giữ nguyên; không tự yêu cầu rollback toàn bộ các field hợp lệ. P15-02/P15-05. Nếu UI ngăn nhập/lưu, ghi điểm chặn và không tự coi backend đã được kiểm tra. | FR15-B-L/2-point/dưới; FR15-B-L/3-point/dưới; EC Domain: FR15-EC-N02 |
| FR15-BVA-WU-N1 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(1), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) bấm Sửa đúng P; (3) điền/nhập thử name=A lặp 1 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Cập nhật được P với name đúng A lặp 1 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên trên UI và khi đọc lại. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/2-point/tại biên; FR15-B-L/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-WU-N2 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(2), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) bấm Sửa đúng P; (3) điền/nhập thử name=A lặp 2 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Cập nhật được P với name đúng A lặp 2 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên trên UI và khi đọc lại. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/3-point/trên; EC Domain: FR15-EC-N01 |
| FR15-BVA-WU-N254 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(254), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) bấm Sửa đúng P; (3) điền/nhập thử name=A lặp 254 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Cập nhật được P với name đúng A lặp 254 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên trên UI và khi đọc lại. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/3-point/dưới; EC Domain: FR15-EC-N01 |
| FR15-BVA-WU-N255 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(255), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) bấm Sửa đúng P; (3) điền/nhập thử name=A lặp 255 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Cập nhật được P với name đúng A lặp 255 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên trên UI và khi đọc lại. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/2-point/tại biên; FR15-B-U/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-WU-N256 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(256), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Mở tab products; (2) bấm Sửa đúng P; (3) điền/nhập thử name=A lặp 256 lần, các trường khác nominal; (4) bấm Lưu nếu control cho phép; (5) ghi giá trị thực trong control, điểm chặn/response và danh sách ngay sau thao tác; (6) refresh/GET products để đọc lại. Không sửa DOM để vượt maxlength/required nếu có. | Không ghi đè name của P bằng name dài 256 ký tự như tên hợp lệ. Các sản phẩm ngoài P vẫn giữ nguyên; không tự yêu cầu rollback toàn bộ các field hợp lệ. P15-02/P15-05. Nếu UI ngăn nhập/lưu, ghi điểm chặn và không tự coi backend đã được kiểm tra. | FR15-B-U/2-point/trên; FR15-B-U/3-point/trên; EC Domain: FR15-EC-N03 |

### Backend API / Create

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| FR15-BVA-KC-N0 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(0), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) POST /api/products, Bearer T_Admin, body B(0); (3) ghi response; (4) GET products để đọc lại sản phẩm mới; không đi qua validation UI. | Không tạo sản phẩm có name rỗng như tên hợp lệ. P15-02. | FR15-B-L/2-point/dưới; FR15-B-L/3-point/dưới; EC Domain: FR15-EC-N02 |
| FR15-BVA-KC-N1 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(1), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) POST /api/products, Bearer T_Admin, body B(1); (3) ghi response; (4) GET products để đọc lại sản phẩm mới; không đi qua validation UI. | Tạo được sản phẩm có name đúng A lặp 1 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/2-point/tại biên; FR15-B-L/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-KC-N2 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(2), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) POST /api/products, Bearer T_Admin, body B(2); (3) ghi response; (4) GET products để đọc lại sản phẩm mới; không đi qua validation UI. | Tạo được sản phẩm có name đúng A lặp 2 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/3-point/trên; EC Domain: FR15-EC-N01 |
| FR15-BVA-KC-N254 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(254), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) POST /api/products, Bearer T_Admin, body B(254); (3) ghi response; (4) GET products để đọc lại sản phẩm mới; không đi qua validation UI. | Tạo được sản phẩm có name đúng A lặp 254 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/3-point/dưới; EC Domain: FR15-EC-N01 |
| FR15-BVA-KC-N255 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(255), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) POST /api/products, Bearer T_Admin, body B(255); (3) ghi response; (4) GET products để đọc lại sản phẩm mới; không đi qua validation UI. | Tạo được sản phẩm có name đúng A lặp 255 lần và dữ liệu hợp lệ. P15-01/P15-02; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/2-point/tại biên; FR15-B-U/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-KC-N256 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; fixture Create độc lập. | Create; B(256), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) POST /api/products, Bearer T_Admin, body B(256); (3) ghi response; (4) GET products để đọc lại sản phẩm mới; không đi qua validation UI. | Không tạo sản phẩm có name dài 256 ký tự như tên hợp lệ. P15-02. | FR15-B-U/2-point/trên; FR15-B-U/3-point/trên; EC Domain: FR15-EC-N03 |

### Backend API / Update

| Test Case ID | Preconditions | Input | Test Steps | Expected Result | Boundary Covered |
| --- | --- | --- | --- | --- | --- |
| FR15-BVA-KU-N0 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(0), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) PUT /api/products/P.id, Bearer T_Admin, body B(0); (3) ghi response; (4) GET products để đọc lại P và mọi sản phẩm khác; không đi qua validation UI. | Không ghi đè name của P bằng name rỗng như tên hợp lệ. Các sản phẩm ngoài P vẫn giữ nguyên; không tự yêu cầu rollback toàn bộ các field hợp lệ. P15-02/P15-05. | FR15-B-L/2-point/dưới; FR15-B-L/3-point/dưới; EC Domain: FR15-EC-N02 |
| FR15-BVA-KU-N1 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(1), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) PUT /api/products/P.id, Bearer T_Admin, body B(1); (3) ghi response; (4) GET products để đọc lại P và mọi sản phẩm khác; không đi qua validation UI. | Cập nhật được P với name đúng A lặp 1 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên khi đọc lại qua GET. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/2-point/tại biên; FR15-B-L/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-KU-N2 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(2), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) PUT /api/products/P.id, Bearer T_Admin, body B(2); (3) ghi response; (4) GET products để đọc lại P và mọi sản phẩm khác; không đi qua validation UI. | Cập nhật được P với name đúng A lặp 2 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên khi đọc lại qua GET. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-L/3-point/trên; EC Domain: FR15-EC-N01 |
| FR15-BVA-KU-N254 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(254), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) PUT /api/products/P.id, Bearer T_Admin, body B(254); (3) ghi response; (4) GET products để đọc lại P và mọi sản phẩm khác; không đi qua validation UI. | Cập nhật được P với name đúng A lặp 254 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên khi đọc lại qua GET. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/3-point/dưới; EC Domain: FR15-EC-N01 |
| FR15-BVA-KU-N255 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(255), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) PUT /api/products/P.id, Bearer T_Admin, body B(255); (3) ghi response; (4) GET products để đọc lại P và mọi sản phẩm khác; không đi qua validation UI. | Cập nhật được P với name đúng A lặp 255 lần và dữ liệu hợp lệ. Q/các sản phẩm khác giữ nguyên khi đọc lại qua GET. P15-01/P15-02/P15-05; không ép mã HTTP/chuỗi thông báo. | FR15-B-U/2-point/tại biên; FR15-B-U/3-point/tại biên; EC Domain: FR15-EC-N01 |
| FR15-BVA-KU-N256 | Admin/T_Admin hợp lệ; C.id tồn tại; nominal theo mục 4; P/Q tồn tại; snapshot danh sách; không phụ thuộc ca trước. | Update P; B(256), chỉ độ dài name thay đổi; price/category/description/imageUrl giữ nominal. | (1) Chuẩn bị snapshot; (2) PUT /api/products/P.id, Bearer T_Admin, body B(256); (3) ghi response; (4) GET products để đọc lại P và mọi sản phẩm khác; không đi qua validation UI. | Không ghi đè name của P bằng name dài 256 ký tự như tên hợp lệ. Các sản phẩm ngoài P vẫn giữ nguyên; không tự yêu cầu rollback toàn bộ các field hợp lệ. P15-02/P15-05. | FR15-B-U/2-point/trên; FR15-B-U/3-point/trên; EC Domain: FR15-EC-N03 |

## 5. Boundary Coverage Matrix

| Boundary ID | Phương pháp / tầng | Giá trị cần bao phủ | Test Cases dùng chung | Trạng thái coverage |
| --- | --- | --- | --- | --- |
| FR15-B-L | 2-point / Web Admin UI / Create | 0 Invalid; 1 Valid | FR15-BVA-WC-N0, FR15-BVA-WC-N1 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-L | 2-point / Web Admin UI / Update | 0 Invalid; 1 Valid | FR15-BVA-WU-N0, FR15-BVA-WU-N1 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-L | 2-point / Backend API / Create | 0 Invalid; 1 Valid | FR15-BVA-KC-N0, FR15-BVA-KC-N1 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-L | 2-point / Backend API / Update | 0 Invalid; 1 Valid | FR15-BVA-KU-N0, FR15-BVA-KU-N1 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-L | 3-point / Web Admin UI / Create | 0 Invalid; 1 Valid; 2 Valid | FR15-BVA-WC-N0, FR15-BVA-WC-N1, FR15-BVA-WC-N2 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-L | 3-point / Web Admin UI / Update | 0 Invalid; 1 Valid; 2 Valid | FR15-BVA-WU-N0, FR15-BVA-WU-N1, FR15-BVA-WU-N2 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-L | 3-point / Backend API / Create | 0 Invalid; 1 Valid; 2 Valid | FR15-BVA-KC-N0, FR15-BVA-KC-N1, FR15-BVA-KC-N2 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-L | 3-point / Backend API / Update | 0 Invalid; 1 Valid; 2 Valid | FR15-BVA-KU-N0, FR15-BVA-KU-N1, FR15-BVA-KU-N2 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 2-point / Web Admin UI / Create | 255 Valid; 256 Invalid | FR15-BVA-WC-N255, FR15-BVA-WC-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 2-point / Web Admin UI / Update | 255 Valid; 256 Invalid | FR15-BVA-WU-N255, FR15-BVA-WU-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 2-point / Backend API / Create | 255 Valid; 256 Invalid | FR15-BVA-KC-N255, FR15-BVA-KC-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 2-point / Backend API / Update | 255 Valid; 256 Invalid | FR15-BVA-KU-N255, FR15-BVA-KU-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 3-point / Web Admin UI / Create | 254 Valid; 255 Valid; 256 Invalid | FR15-BVA-WC-N254, FR15-BVA-WC-N255, FR15-BVA-WC-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 3-point / Web Admin UI / Update | 254 Valid; 255 Valid; 256 Invalid | FR15-BVA-WU-N254, FR15-BVA-WU-N255, FR15-BVA-WU-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 3-point / Backend API / Create | 254 Valid; 255 Valid; 256 Invalid | FR15-BVA-KC-N254, FR15-BVA-KC-N255, FR15-BVA-KC-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-U | 3-point / Backend API / Update | 254 Valid; 255 Valid; 256 Invalid | FR15-BVA-KU-N254, FR15-BVA-KU-N255, FR15-BVA-KU-N256 | Đủ các điểm yêu cầu trong thiết kế; chưa thực thi. |
| FR15-B-P0 | Chưa đủ cơ sở | 0 đã biết; lân cận chưa xác định | Không tạo Test Case BVA mới | Không tính là bộ BVA chuẩn đã thiết kế. |

Các ID trong matrix tham chiếu đúng một dòng Test Case vật lý. Không cộng trùng Test Case giữa 2-point và 3-point; số ca đã thực thi là 0.

## 6. Giả định, giới hạn và các điểm cần human review

1. Diễn giải required là name không rỗng được Task 03 cho phép; dùng ASCII A để tránh tự quyết định Unicode/trim. L=1 chỉ là biên độ dài trong tập tên có nội dung, không khẳng định một dấu cách là tên hợp lệ. Không tạo độ dài âm.
2. SRS chưa quy định price precision/step hoặc max; điểm cắt 0 đã nhận diện nhưng coverage 2-point/3-point price chưa xác định được. Không báo toàn bộ price đã có BVA coverage.
3. Chuỗi số/kiểu JSON và partial Update chưa được mô tả đầy đủ. Các ca chỉ đổi name và gửi body đầy đủ hợp lệ còn lại; không phân lớp lỗi kiểu JSON mới.
4. UI có thể ngăn nhập hoặc lưu trước khi gửi request. Khi chạy sau này phải ghi giá trị thực và điểm chặn, không thay ca UI bằng API; các ca API độc lập không bị che bởi UI.
5. Không tạo BVA cho category/product ID từ trị số ID, không suy ra giới hạn description/imageUrl. Read/Delete chỉ dùng làm quan sát hoặc thuộc Domain Testing, không cần bộ name-boundary giả.
6. Tính độc lập Update được đối chiếu cả UI ngay sau lưu và GET/refresh; không kết luận từ source rằng database đã bị sửa hàng loạt.

Đã hoàn thành thiết kế Task 03 cho chức năng này. Dừng để human review; chưa chuyển sang Test Execution.
