# Event ở Admin hiển thị nổi bật trên Client

## Phần đã triển khai

- **Dữ liệu dùng chung**: Admin tạo Event / revision trên cùng Backend JSON với Client, không dùng `USE_MOCK`.
- **Tự chạy theo ngày/giờ**: `GET /api/v1/shop/events` chỉ công khai Event đang bật, có revision `APPROVED`, `startAt <= now < endAt`. Event nháp, chờ duyệt, chưa đến ngày, kết thúc, đã disable hoặc soft-delete không được phát ra API Client.
- **Revision đã lên lịch**: nếu revision được duyệt với `scheduledRevNo` và thời điểm bắt đầu đã đến, Client sử dụng revision mới cả cho UI lẫn giá thanh toán, dù chưa có ai mở Admin.
- **Giao diện thay đổi theo Event**: top ribbon toàn website, thanh điều hướng Sự kiện, hero lớn thay hero thường trên trang chủ, nội dung giới thiệu, CTA, số ưu đãi, sách giảm giá, combo, thời gian & đồng hồ đếm ngược. Đổi màu theo `FOREST`, `TET`, `SUNSET` hoặc `MIDNIGHT`.
- **Trang Event**: `http://localhost:3000/#/events` (Event đang chạy) và `http://localhost:3000/#/events/ID`. Có thể truy cập từ ribbon và nút xem sự kiện của Admin.
- **Hiệu lực giá**: Backend quyết định `salePrice`, `currentPrice`, `quote` và tổng tiền; Client chỉ đọc. Nếu Event bị tắt/hết hạn, quote và trang sách dùng lại giá không giảm.
- **Tự đồng bộ**: khi Client đang mở, thăm dò API mỗi 30 giây, khi trở lại tab và mỗi khi chuyển trang; đồng hồ đếm ngược được cập nhật mỗi giây. Không cần nhân viên reload trang sau khi một Event được duyệt, nhưng có thể có độ trễ tối đa khoảng 30 giây khi tab đang mở.
- **An toàn**: dữ liệu Event chưa được duyệt không xuất hiện trong API public; các trường text được escape chống HTML injection; theme được whitelist; API không cung cấp thông tin duyệt nội bộ.

## 1. Tạo Event và chạy thử trên Admin

1. Mở `http://localhost:3000/admin/` và đăng nhập Manager (`manager@nhasach.test` / `Manager@123`) ở dữ liệu demo mới.
2. Chọn **Khuyến mãi → Event** (mục Event bên trái) → **+ Tạo Event**.
3. Nhập:
   - **Mã Event**: `BOOK_WEEKEND_01`.
   - **Tên Event**: `Hội sách cuối tuần`.
   - **Loại**: `Book Sale` hoặc `Tết Sale`.
   - **Tiêu đề banner**: `Cuối tuần đọc hay, giá mê say`.
   - **Mô tả**: `Ưu đãi những cuốn sách chọn lọc cho cuối tuần thư giãn.`
   - **Nhãn nổi bật**: `SALE CUỐI TUẦN`.
   - **Giao diện / màu sắc**: `Cam đào` hoặc màu bạn muốn.
   - **Bắt đầu**: vài phút **trước** thời điểm hiện tại; **Kết thúc**: vài ngày **sau** thời điểm hiện tại. Chú ý múi giờ hệ thống Windows. Không chọn lịch trùng Event đã duyệt còn hoạt động.
   - **Sách giảm giá**: nhấn `+ Thêm sách giảm giá`, chọn sách đang được xuất bản, nhập giá sale nhỏ hơn giá gốc.
4. Nhấn **Lưu nháp** → mở Event → **Gửi duyệt** → đăng nhập Manager và nhấn **Duyệt**.
5. Trong danh sách Event, khi đúng thời gian sẽ có nhãn **Đang hiển thị Client**. Mở chi tiết Event → **Mở Event trên Client** hoặc truy cập trang chủ `http://localhost:3000/`.
6. Nếu muốn chỉnh copy/theme cho Event đã duyệt, chọn **+ Tạo revision mới**, sửa, gửi duyệt và phê duyệt revision đó; không sửa trực tiếp revision đã duyệt.

> Dữ liệu mặc định có `BOOK_SALE` chạy ngay để bạn thấy banner. Event `TET_SALE` được đặt lịch tương lai. Nếu tạo Event chạy ngay và bị báo `EVENT_OVERLAP`, hãy tắt Event mẫu trước (chỉ trên máy lab) hoặc chọn khung giờ không trùng, rồi gửi duyệt lại.

## 2. Luồng của Client

- **Có Event chạy**: `/#/` hiển thị banner theo chủ đề, tên Event, lời giới thiệu, giảm tối đa (nếu có), đếm ngược, nút khám phá Event, sản phẩm giảm giá. Header hiện ribbon trên **mọi** trang.
- **Nhấn “Khám phá ưu đãi”**: mở `/#/events/ID` với đầy đủ sách giảm giá và combo thuộc Event.
- **Nhấn mua**: giỏ hàng và thanh toán dùng đúng giá do Backend tính, bao gồm mức giảm Event.
- **Chưa có hoặc hết Event**: về giao diện hero Nhà sách bình thường; ribbon và liên kết sự kiện được ẩn. Vào `/#/events` sẽ thấy thông báo chưa có Event đang diễn ra.
- **Event có sách bị ngừng bán**: sách đó không hiển thị; Backend không áp dụng giá sale cho sách không bán được.

## 3. API

| API | Quyền | Kết quả |
| --- | --- | --- |
| `GET /api/v1/shop/events` | Công khai | `{items:[...],serverNow}`. Chỉ sự kiện đang diễn ra. |
| `GET /api/v1/shop/events/:id` | Công khai | Chi tiết 1 Event đang diễn ra, `404` nếu chưa chạy/hết hạn/disable. |
| `GET /api/v1/shop/home` | Công khai | Có thêm `activeEvents`, dùng cho banner trang chủ. |
| `GET /api/v1/shop/books/:id` | Công khai | Sách giảm giá có `promotionEvent: {rowId,name}` nếu đang tham gia. |
| `POST /api/v1/shop/checkout/quote` | Buyer tùy chính sách | Giá/tổng tiền trên server theo revision Event đang chạy. |

Event public: `rowId`, `code`, `name`, `type`, `theme`, `badgeText`, `headline`, `description`, `startAt`, `endAt`, `maxDiscountPercent`, `bookCount`, `bundleCount`, `books[]`, `bundles[]`. Không trả dữ liệu revisions nội bộ.

## 4. Giữ dữ liệu bản cũ khi nâng cấp

1. Dừng server cũ bằng `Ctrl+C`.
2. Sao lưu `backend/data/db.json` **và toàn bộ** `backend/storage/` (ảnh bìa, PDF...), nếu đã nhập dữ liệu. Tránh ghi đè dữ liệu mới mà chưa backup.
3. Giải nén phiên bản này sang một thư mục mới. Chép dữ liệu đã sao lưu về `backend/data/db.json` và `backend/storage/` tương ứng trong project mới.
4. Mở PowerShell ở `webBookStore-main` và chạy `npm.cmd start`.
5. Mở `/admin/` → Event → chọn một Event đã duyệt → **Tạo revision mới** để bổ sung banner, mô tả và theme. Các Event cũ không có trường này vẫn hiển thị bằng **tên Event** và theme tự động.
6. Mở Client tại `http://localhost:3000/`, kiểm tra sự kiện và giá trong giỏ hàng.

**Lưu ý:** bản này phục vụ lab / demo; thanh toán là demo. Muốn chạy website liên tục 24/7 cần triển khai Node.js trên server luôn hoạt động. Đồng hồ đếm ngược dựa trên giờ máy khách, hiệu lực thật và giá luôn do thời gian Backend quyết định.

## 5. Kiểm thử

Chạy `npm.cmd test`. Bộ thử tự động kiểm tra (a) Event active hiển thị và giá quote, (b) Event pending/draft/future không lộ, (c) disable đưa giá về bình thường, (d) tạo → submit → approve → hiển thị, (e) scheduled revision, (f) chống HTML injection, (g) sự kiện trên trang chủ và trang Event, ngoài 7 bộ thử cũ.
