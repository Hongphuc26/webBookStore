# Quản lý ảnh bìa sách – Admin → Backend → Client

## Hướng dẫn thao tác trên Windows

1. Sao lưu dự án cũ: **dừng server** (`Ctrl+C`), sao chép `backend/data/db.json` và **toàn bộ** thư mục `backend/storage/` (gồm PDF và `covers/` nếu có) ra nơi an toàn.
2. Giải nén bản ZIP có tính năng ảnh bìa. Nếu bạn đã nhập sách/đơn hàng trên bản trước, hãy chép lại `db.json` cùng thư mục `storage/` vào đúng vị trí tương ứng trong bản mới, không chép đè lên dữ liệu hiện có khi chưa sao lưu. Không cần tạo lại Admin/Client hay reset tài khoản.
3. Mở PowerShell trong thư mục `webBookStore-main` và chạy `npm.cmd start` (Node.js >= 20).
4. Mở `http://localhost:3000/admin/`, đăng nhập Manager hoặc Employee → **Sách & PDF**.
5. Nhấn một dòng sách bất kỳ → **Tải ảnh bìa** hoặc **Thay / xóa ảnh bìa**; cũng có thể nhấn **+ Thêm sách** hoặc **Sửa thông tin**.
6. Ở cuối form **Thêm/Sửa sách** có khung **Ảnh bìa sách**. Chọn file JPG/JPEG/PNG/WebP, xem ảnh mới xuất hiện trong khung bên trái → **Lưu sách**.
7. Mở `http://localhost:3000/` → trang chủ, Khám phá sách, chi tiết sách, giỏ hàng, danh sách trong Combo hoặc chatbot tư vấn sách. Nếu sách đang ở `PUBLISHED` và `ACTIVE`, ảnh mới sẽ xuất hiện. Tải lại trang (`Ctrl+Shift+R`) nếu ảnh cũ vẫn đang trong cache.
8. Muốn xóa ảnh: vào **Sửa thông tin** → **Xóa ảnh** → **Lưu sách**. Sách về bìa CSS mặc định. Nhấn **Hủy** sẽ không thay đổi ảnh lưu trên server.

**Lưu ý:** Upload ảnh bìa là thao tác riêng, không thay thế file PDF. Sách nháp, ẩn, đình chỉ phục vụ không có ảnh bìa công khai trên Client. Chỉ khi đã có PDF active thì Admin mới đổi sách thành `PUBLISHED` (quy tắc cũ, không thay đổi).

## Kiến trúc

```text
ADMIN                          BACKEND (Node.js)                  CLIENT
Sách & PDF                      /api/v1                            Trang chủ / Chi tiết / Chatbot
Chọn JPG/PNG/WebP   --POST-->   /admin/books/:id/cover             GET /shop/books
Thay hoặc Xóa       --DELETE--> (role=EMPLOYEE/MANAGER, If-Match)  -> coverUrl
      |                              |                            GET /shop/books/:id/cover
      |                       data/db.json                        Chỉ sách public
      |                       storage/covers/*.jpg/png/webp
      +--- GET /admin/books/:id/cover (Bearer Token, kể cả sách DRAFT)
```

Metadata (`cover`) được lưu **trong cùng record sách** của DB. File thật được lưu bên dưới `backend/storage/covers/` theo UUID để tránh trùng tên/path traversal. GET Client không nhận `storageKey`, chỉ nhận URL động theo ID và version. Khi Admin thay ảnh, file cũ bị xóa sau khi bản mới được ghi thành công; khi xóa ảnh, file cũ cũng được dọn.

- Định dạng cho phép: **JPG, JPEG, PNG, WebP**. Không nhận SVG/GIF/HTML hoặc file giả mạo MIME thông thường.
- Kích thước tối đa: **5MB/ảnh**, tối đa 6000 px mỗi cạnh, 24 megapixel. Tỷ lệ dọc 2:3 là gợi ý, không bắt buộc. Ảnh gốc không bị cắt trên Client (`object-fit: contain`).
- Authorization: Admin cần phiên Bearer với `BOOK_WRITE`; Buyer không thể tải, thay, xóa hoặc xem cover nháp qua Admin API.
- Tránh ghi đè khi hai người cùng thao tác: `If-Match` buộc khớp `book.version`; sai phiên bản trả `409 VERSION_CONFLICT`.
- File upload được server kiểm tra header định dạng, kích thước, MIME. Đây không phải dịch vụ antivirus / xử lý hình ảnh đầy đủ; muốn đưa ra production cần dùng image-decoding/transcoding, quét malware, hạn mức storage và CDN.
- Nếu chưa có cover hoặc ảnh không tải được, Client giữ bìa CSS minh họa cũ. Chế độ `USE_MOCK=true` không hỗ trợ upload file ảnh thật.

## Triển khai & backup

`docker-compose.yml` đã gắn volume `bookstore_pdf` vào **toàn bộ** `/app/backend/storage`, vì vậy ảnh trong `/app/backend/storage/covers/` cũng được giữ lại giữa các lần chạy. Khi chạy thuần Node, cần sao lưu **hai thành phần cùng nhau**:

- `backend/data/db.json`: metadata, ảnh bìa của từng sách, order, user...
- `backend/storage/`: tất cả PDF và ảnh bìa thực.

Có thể đổi nơi lưu bằng biến `COVER_DIR`, nhưng nhớ backup thư mục được cấu hình mới. Không công khai thư mục `storage` bằng static file server.

## Kiểm thử

Chạy `npm.cmd test` (Windows PowerShell) hoặc `npm test` (macOS/Linux). File `backend/tests/book-covers.test.js` kiểm tra upload ảnh thật, thay ảnh, xóa ảnh, quyền Admin/Buyer, lỗi file giả, giới hạn 5MB, phiên bản If-Match, trạng thái public/draft, ảnh trong API Client, và chatbot; test tạo DB/storage tạm, không đụng dữ liệu lab.
