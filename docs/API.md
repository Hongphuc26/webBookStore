# API v1 — hợp đồng chung của Client và Admin

**Base URL:** `http://localhost:3000/api/v1` (mặc định). Admin, Client và Backend cùng một origin. Request và response dùng JSON UTF-8 trừ các endpoint upload/download PDF và ảnh bìa (binary).

Các endpoint yêu cầu xác thực dùng `Authorization: Bearer <accessToken>`.

Quyền:
- `Public`: không cần token.
- `Buyer`: token tài khoản `role=BUYER`.
- `Staff`: token `role=EMPLOYEE` hoặc `MANAGER`, phải có permission cụ thể.

Lỗi có dạng:

```json
{
  "errorCode": "FORBIDDEN",
  "message": "Không có quyền truy cập",
  "details": null,
  "requestId": "a-random-request-id"
}
```

## 1. Authentication (cùng bảng Users và Sessions)

| Method | Route | Quyền | Mô tả |
|---|---|---|---|
| POST | `/auth/register` | Public | Tạo Buyer mới (`email`, `fullName`, `password`) |
| POST | `/auth/login` | Public | Đăng nhập (`email`, `password`, `audience: "BUYER"` hoặc `"STAFF"`) |
| GET | `/auth/me` | Auth | Trả `user` và `permissions` |
| PATCH | `/auth/me` | Auth | Cập nhật `fullName` |
| POST | `/auth/logout` | Auth | Hủy phiên hiện tại |

Ví dụ đăng ký:

```http
POST /api/v1/auth/register
Content-Type: application/json

{"fullName":"Nguyễn Khách Hàng","email":"buyer@example.com","password":"MyPassword123"}
```

`201 Created`:

```json
{
  "accessToken": "<bearer-token>",
  "user": { "rowId": 2120, "fullName": "Nguyễn Khách Hàng", "email": "buyer@example.com", "role": "BUYER", "status": "ACTIVE", "version": 1 },
  "permissions": []
}
```

`rowId` và token trong ví dụ chỉ minh họa, giá trị thực thay đổi theo DB.

## 2. Client public — catalog

| Method | Route | Quyền | Mô tả |
|---|---|---|---|
| GET | `/shop/home` | Public | Sách đề xuất, chủ đề và bundle active |
| GET | `/shop/categories` | Public | Danh mục có số lượng sách bán |
| GET | `/shop/books` | Public | `q`, `categoryId`, `sort`, `page`, `pageSize` |
| GET | `/shop/books/:id` | Public | Chi tiết sách, review visible, `coverUrl` khi có ảnh |
| GET/HEAD | `/shop/books/:id/cover` | Public | Bytes ảnh bìa nếu sách PUBLISHED + ACTIVE (nếu không 404) |
| GET | `/shop/bundles` | Public | Bundle được duyệt, còn hoạt động |
| GET | `/shop/bundles/:id` | Public | Combo và danh sách sách trong combo |
| GET | `/shop/coupons` | Public | Coupon đã duyệt, còn hiệu lực |
| POST | `/shop/checkout/quote` | Public/Buyer | Tính báo giá từ DB, **không tin dữ liệu price từ client** |

`GET /shop/books?q=code&sort=price_asc&page=1&pageSize=12` trả:

```json
{
  "items": [{
    "rowId": 1, "bookId": "BK001", "title": "Mắt biếc",
    "authorNames": ["Nguyễn Nhật Ánh"], "categoryId": 1, "categoryName": "Văn học",
    "price": 60000, "salePrice": 48000, "currentPrice": 48000,
    "hasPdfMetadata": true, "canDownload": true, "pdfSource": "DEMO", "coverUrl": null, "rating": 0, "reviewCount": 0
  }],
  "total": 1, "page": 1, "pageSize": 12
}
```

Các giá trị/quan hệ trong ví dụ là định dạng minh họa; kết quả thực lấy từ DB seed.

### Cart / Quote

Client lưu giỏ trong `localStorage` và gửi item ID/số lượng. Backend tự tra giá, event giảm giá, trạng thái sách, combo và coupon. API không cung cấp server-side cart persistence trong bản lab.

```http
POST /api/v1/shop/checkout/quote
Content-Type: application/json

{
  "items": [
    {"type":"BOOK","id":1,"quantity":1},
    {"type":"BUNDLE","id":1,"quantity":1,"selectedBookIds":[1,2,3,10]}
  ],
  "couponCode":"CP10"
}
```

Với Bundle `FIXED`, danh sách chọn được tính tự động từ revision đã duyệt. Với Bundle `FLEXIBLE`, Client phải gửi `selectedBookIds` đúng `pickCount` và nằm trong các sách hợp lệ của combo.

Response:

```json
{
  "items": [{"type":"BOOK","bookRowId":1,"name":"Mắt biếc","quantity":1,"unitPrice":48000,"price":48000,"bookIds":[1]}],
  "subtotal":60000,
  "eventDiscount":12000,
  "couponDiscount":4800,
  "discount":16800,
  "couponCode":"CP10",
  "total":43200
}
```

Ví dụ response mô tả cách tính cho một item; phần `items` và số liệu thực phải phụ thuộc giỏ hàng gửi lên.

## 3. Client private — Buyer

| Method | Route | Quyền | Mô tả |
|---|---|---|---|
| POST | `/shop/orders` | Buyer | Tạo đơn `PENDING_PAYMENT` sau khi tự báo giá |
| GET | `/shop/orders` | Buyer | Đơn do Buyer tạo |
| GET | `/shop/orders/:id` | Buyer owner | Chi tiết đơn; không xem được đơn người khác |
| POST | `/shop/orders/:id/demo-pay` | Buyer owner, lab | Chuyển trạng thái `PAID` **không thu tiền thật** |
| POST | `/shop/orders/:id/cancel` | Buyer owner | Hủy đơn đang `PENDING_PAYMENT` |
| GET | `/shop/library` | Buyer | Sách theo đơn PAID mua thường hoặc tặng đã nhận |
| POST | `/shop/gifts/claim` | Buyer | Nhận sách bằng `{ "code": "GFT-..." }` và email người nhận |
| GET | `/shop/gifts/received` | Buyer | Quà đã nhận từ tài khoản khác |
| GET | `/shop/library/:id/pdf` | Buyer có entitlement | Stream PDF thật hoặc bản demo Lorem Ipsum (binary) |
| GET | `/shop/favorites` | Buyer | Danh sách yêu thích |
| POST | `/shop/favorites/:id` | Buyer | Lưu sách |
| DELETE | `/shop/favorites/:id` | Buyer | Bỏ lưu |
| POST | `/shop/reviews` | Buyer đã mua | Đánh giá sao/nội dung (`bookId`, `rating`, `content`) |
| POST | `/shop/content-reports` | Buyer | Báo cáo sách/review (`targetType`, `targetRowId`, `reason`) |
| POST | `/shop/complaints` | Buyer owner đơn | Tạo khiếu nại (`orderRowId`, `subject`, `message`) |
| GET | `/shop/complaints` | Buyer | Danh sách yêu cầu của mình |
| GET | `/shop/complaints/:id` | Buyer owner | Đọc tin nhắn, **lọc bỏ `internal:true`** |
| POST | `/shop/complaints/:id/messages` | Buyer owner | Gửi tin nhắn (`body`) |

Order flow:

```http
POST /api/v1/shop/orders
Authorization: Bearer <buyer-token>
Content-Type: application/json

{"items":[{"type":"BOOK","id":1,"quantity":1}],"couponCode":""}
```

Phản hồi `201` là một `Order` với `rowId`, `orderId`, `status: "PENDING_PAYMENT"`, `items`, `subtotal`, `discount`, `total`, `payment`, `history`.

Trong lab, `POST /api/v1/shop/orders/:id/demo-pay` chuyển đơn sang `PAID`; đơn mua thường cấp quyền ngay, đơn tặng **chỉ cấp quyền cho người nhận sau khi họ nhập mã**. **Không dùng thao tác này trên Internet/production.** Đối với backend thật, trạng thái PAID phải do callback/webhook đã kiểm chữ ký và đối soát thanh toán cập nhật.

### Tạo và nhận quà tặng (Client)

Người mua tạo đơn tặng bằng cách thêm object `gift` vào `POST /shop/orders`:

```json
{
  "items":[{"type":"BOOK","id":2,"quantity":1}],
  "couponCode":"",
  "gift":{"recipientEmail":"friend@example.com","recipientName":"Bạn thân","message":"Đọc sách vui nhé!"}
}
```

- Backend xác thực email, không cho tự tặng chính mình, hạn chế mỗi dòng sản phẩm quà tặng có số lượng 1, tự tính lại giá. Có thể tặng cả combo đang bán.
- Khi đơn `PENDING_PAYMENT`: response `gift.claimCode = null`. Sau `demo-pay` thành công, người mua xem mã `gift.claimCode` trong `GET /shop/orders/:id` hoặc phản hồi thanh toán. Không lưu mã rõ trong DB; mã được ký HMAC bằng khóa được lưu ở DB (giữ nguyên khi sao lưu/khôi phục DB).
- Người nhận đăng ký/đăng nhập đúng email đã nhập và `POST /shop/gifts/claim` với `{ "code":"GFT-..." }`. Hệ thống kiểm tra thanh toán, email, chữ ký mã và người đã nhận; lưu `recipientUserRowId`, sau đó mới cấp quyền đọc. `GET /shop/gifts/received` trả quà đã nhận. Người gửi không được quyền đọc sách tặng; người nhận không xem hóa đơn của người gửi.
- Chưa có email notification, email verification, hoặc thanh toán thật. Có hạn chế thử mã và audit trong Backend.

### Đọc sách PDF ngay trên website

`GET /api/v1/shop/library/:id/pdf?view=1` (có Bearer token) trả `application/pdf`, `Content-Disposition:inline`, `X-Book-Content: demo|uploaded`. Không có `view=1` thì `attachment` để tải. Với sách chưa upload PDF thật, module `demoPdf.js` sinh **PDF 7 trang** từ nội dung Lorem Ipsum vô hại, ghi rõ đó là file minh họa, không sử dụng nội dung của tác phẩm gốc. Trang Client dùng `fetch` với Bearer token rồi tạo Blob URL trong tab, không đặt token trong URL iframe. Không phát PDF cho khách chưa có quyền.

## 4. Admin — giữ hợp đồng API gốc

`backend/src/adminRoutes.js` kế thừa logic của `frontend/admin/js/mock/server.js` và tiếp tục xử lý các endpoint:

| Nhóm | Route quan trọng |
|---|---|
| Books | `GET /admin/books`, `POST /admin/books`, `PATCH /admin/books/:id`, `POST /admin/books/:id/serving-status` |
| PDF (mới) | `POST /admin/books/:id/pdf` — **binary application/pdf** (50MB) |
| Cover | `GET/HEAD /admin/books/:id/cover`, `POST /admin/books/:id/cover`, `DELETE /admin/books/:id/cover` — STAFF `BOOK_WRITE` |
| Catalog | `/admin/authors`, `/admin/categories`, `/admin/publishers` |
| Revisions | `/admin/bundles`, `/admin/events`, `/admin/coupons` và `/revisions/:rev/:action` |
| Orders | `GET /admin/orders`, `GET /admin/orders/:id` |
| Users/Staff | `/admin/users`, `/admin/staff`, disable/enable |
| Complaints | `/admin/complaints`, `POST /complaints/:id/messages` |
| Moderation | `/admin/content-reports`, trạng thái phục vụ sách |
| Reports/Audit | `/admin/reports/operational`, `/admin/reports/financial`, `/admin/audit-logs` |
| Other | `/admin/history`, `/admin/knowledge-articles`, `/admin/settings`, `/ai/admin/chat` |

`PATCH` các entity có version yêu cầu `If-Match: <version>` để bắt xung đột (HTTP 409). `POST /admin/books/:id/files` từ contract cũ vẫn chỉ ghi metadata; nên dùng endpoint PDF mới để cung cấp nội dung đọc/tải thật.

### Ảnh bìa sách (JPG/PNG/WebP)

- **POST** `/admin/books/:id/cover` — **STAFF** `BOOK_WRITE`. Raw body bytes của ảnh; `Content-Type: image/jpeg` hoặc `image/png` hoặc `image/webp`; `X-File-Name: <encodeURIComponent(filename)>`; `If-Match: <book.version>`; tối đa 5MB, kiểm tra magic/dimensions. `201` trả `{rowId,bookId,version,hasCover,coverUrl,coverUpdatedAt}`. Khi thay, xóa file cũ sau khi lưu metadata mới.
- **DELETE** `/admin/books/:id/cover` — **STAFF** `BOOK_WRITE`, có `If-Match: <book.version>`. Xóa ảnh, cập nhật version, `coverUrl:null`. Dùng `GET /admin/books/:id` để đọc version mới nhất trước khi gọi.
- **GET/HEAD** `/admin/books/:id/cover` — **STAFF** `BOOK_WRITE`. Trả binary với `Content-Type` của ảnh để xem trước cả sách nháp/ẩn. Frontend Admin dùng Bearer `fetch()` và Blob URL, **không** nhúng token vào URL.
- **GET/HEAD** `/shop/books/:id/cover` — **PUBLIC**, chỉ trả ảnh bìa nếu sách `PUBLISHED`, `ACTIVE`, không xóa mềm và có ảnh thực. Ngược lại 404. `coverUrl` trong response book public là `/api/v1/shop/books/:id/cover?v=<version>` hoặc `null`; không lộ `storageKey` hay đường dẫn nội bộ.

`POST/DELETE` kiểm tra `If-Match` và trả 428 (thiếu version), 409 (stale). 415 MIME sai, 422 file giả/không hợp lệ, 413 quá 5MB. Chi tiết thao tác tại [BOOK_COVERS.md](BOOK_COVERS.md).

### Upload PDF thật

```http
POST /api/v1/admin/books/29/pdf
Authorization: Bearer <staff-token>
Content-Type: application/pdf
X-File-Name: mybook.pdf

<binary bytes của file PDF>
```

Backend kiểm quyền staff, dung lượng ≤ 50 MB, header PDF, tạo `storageKey` UUID tại `backend/storage/`, ghi metadata/version vào DB và hủy active file cũ. Client chỉ nhận nội dung tại `GET /shop/library/:id/pdf` khi Buyer có quyền từ đơn PAID mua thường hoặc tặng đã nhận và sách còn phục vụ. Thêm `?view=1` để trả Content-Disposition inline; không có file upload thì sinh PDF demo 7 trang. Cả hai cách đều yêu cầu Bearer token và không công khai đường dẫn file.

## 5. Khi đổi sang Backend của nhóm

Giữ nguyên những đặc tính sau để cả Admin và Client hoạt động:

1. Cùng đường dẫn `/api/v1`, cùng `Users`, `Books`, `Bundles`, `Events`, `Coupons`, `Orders`, `Reviews`, `Complaints`.
2. Access token có phân biệt role và kiểm quyền **tại server**.
3. `bookId` khác `rowId`, `orderId` khác `rowId`; Client dùng rowId cho route và liên kết DB.
4. Coupon & Event chạy qua revision được duyệt, không hiển thị draft.
5. Tính giá, giảm giá, total trên server; callback thanh toán có bảo vệ.
6. Đơn đã thanh toán cấp entitlement theo buyer/recipient, PDF tải qua API có kiểm quyền.
7. `GET /admin/orders` nhìn thấy chính các đơn do Client tạo, `/admin/users` nhìn thấy tài khoản Buyer đã đăng ký.
8. Không để `USE_MOCK: true` trên Admin khi cần kiểm tra dữ liệu chung.


---

## Chatbot Client: sách + phạm vi giới hạn (mới)

### `GET /api/v1/shop/chat/status` — công khai

```json
{ "available": true, "mode": "catalog", "scope": "BOOKSTORE_ONLY", "requiresLogin": false }
```

`mode: "ai"` nghĩa là Backend đã cấu hình `OPENAI_API_KEY`, **không** bảo đảm dịch vụ AI đang hoạt động. Từng câu trả lời sẽ báo `mode` thực tế.

### `POST /api/v1/shop/chat` — công khai

`Content-Type: application/json`; rate limit tối đa 15 request/IP/phút; `message` chuỗi 1–400 ký tự; `contextBookIds` tùy chọn, 0–3 ID sách đã được hiển thị cho khách để hỏi tiếp về giá/tác giả. Backend tự kiểm chứng sách phải công khai.

```json
{ "message": "Mắt biếc giá bao nhiêu?", "contextBookIds": [] }
```

Ví dụ phản hồi (giá theo dữ liệu DB hiện tại):

```json
{
  "answer": "Giá hiện tại của “Mắt biếc” là 48.000đ (giá gốc 60.000đ). Tác giả: Nguyễn Nhật Ánh. Bạn có thể mở chi tiết sách ngay bên dưới.",
  "books": [{"rowId":1,"title":"Mắt biếc","authorNames":["Nguyễn Nhật Ánh"],"categoryName":"Văn học","price":60000,"currentPrice":48000,"salePrice":48000,"coverUrl":null,"url":"#/book/1"}],
  "inScope": true,
  "mode": "catalog",
  "suggestions": ["Gợi ý sách văn học", "Sách nào đang giảm giá?", "Cách mua sách PDF?"]
}
```

Ngoài phạm vi luôn trả `inScope:false`, `mode:"restricted"`, `books:[]` và **một thông báo từ chối ngắn**. `400`/`422` được dùng cho dữ liệu không hợp lệ, `429` khi quá lượt/phút. Câu trả lời không dùng nội dung sách nháp, sách ngừng bán, kiến thức Admin nội bộ, lịch sử Buyer hay những trường ngoài danh mục công khai.

### Admin knowledge public flag

`POST/PATCH /api/v1/admin/knowledge-articles`: field mới `clientVisible: boolean` (mặc định `false`). Manager phải đặt **`status:"PUBLISHED"` và `clientVisible:true`** thì bài hướng dẫn mới được chatbot Client truy xuất. Chuyển sang `DRAFT` hoặc bỏ `clientVisible` sẽ ngăn truy xuất ngay lượt hỏi kế tiếp. Không đưa bài viết chứa dữ liệu mật vào phạm vi Client.

### AI tích hợp tùy chọn

`OPENAI_API_KEY` chỉ được đọc trong Node Backend. Gọi `POST /chat/completions` với `response_format: {"type":"json_object"}`; LLM **chỉ được phép phân loại chủ đề và chọn ID** từ danh mục hiện đang public. Backend xác thực lại ID, tự biên soạn giá/chi tiết từ DB. Nội dung tự do từ LLM không được gửi thẳng cho khách. Kể cả khi có AI, bộ lọc phạm vi ở Backend vẫn từ chối các yêu cầu ngoại chủ đề trước khi gọi model.

## Sự kiện khuyến mãi ở Client

- `GET /shop/events`: danh sách Event `APPROVED` đang chạy, gồm banner copy, theme, thời gian, sách/combo giảm giá.
- `GET /shop/events/:id`: chi tiết Event đang chạy hoặc `404` nếu chưa chạy/hết hạn.
- `GET /shop/home`: có thêm `activeEvents` để hiển thị hero đang chạy.
- `GET /shop/books/:id`: có thêm `promotionEvent` nếu sách đang được Event giảm giá.
- Giá thật của Event vẫn do Backend quyết định khi `POST /shop/checkout/quote` và tạo order.

Chi tiết trong `docs/EVENTS_CLIENT.md`.
