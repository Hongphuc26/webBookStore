# Checklist demo Admin ↔ Client

**Chuẩn bị:** chạy `npm start` ở thư mục gốc, mở hai tab ở cùng trình duyệt:
- Tab A: `http://localhost:3000/admin/`
- Tab B: `http://localhost:3000/`

## Luồng 1 — Đăng sách và xem ở Client

1. Admin đăng nhập `manager@nhasach.test / Manager@123`.
2. Chọn **Sách & PDF → + Thêm sách**: BookId `TEST001`, tên `Kiểm thử đồng bộ`, chọn Tác giả, Danh mục, giá `99000` → Lưu. Trạng thái DRAFT.
3. Mở chi tiết sách mới → **+ Tải lên phiên bản PDF** → chọn một file PDF hợp pháp có nội dung thực. Backend sẽ kiểm định PDF và lưu vào storage.
4. Sửa sách → `PUBLISHED` → Lưu.
5. Client vào **Khám phá sách**, tìm `Kiểm thử đồng bộ`. Sách phải xuất hiện với đúng giá/description.
6. Quay lại Admin, đổi giá → Client tải lại và thấy giá mới (không lưu giá trong frontend).

## Luồng 2 — Giỏ hàng, thanh toán thử và đối soát

1. Client đăng ký Buyer mới hoặc đăng nhập `an@mail.test / Buyer@123`.
2. Chọn sách `TEST001` → Thêm vào giỏ → Giỏ hàng → Tạo đơn hàng.
3. Trong trang chi tiết đơn, nhấn **Xác nhận thanh toán demo**. Màn hình báo **Đã thanh toán**. Không có tiền thật được xử lý.
4. Admin → **Đơn hàng (hỗ trợ)**: thấy đơn mới `ORD-...`, đúng Buyer và tổng tiền. **Báo cáo** cũng cập nhật.
5. Client → **Thư viện của tôi**, tìm sách vừa mua → nhấn **Tải bản PDF**. Kiểm tra file tải xuống.
6. Đăng nhập Buyer khác: không được tải PDF của Buyer vừa mua (API trả 403).

## Luồng 3 — Bundle, Event, Coupon

1. Admin → Bundle/Event/Coupon: tạo revision DRAFT theo giao diện cũ, gửi duyệt.
2. Chuyển tài khoản Manager để duyệt revision.
3. Client: combo mới hiển thị trong Combo sách; Event active làm thay đổi giá sale; Coupon còn hạn áp dụng được vào checkout.
4. Coupon hết hạn/không đạt minSpend/hết quota phải trả lỗi từ server. Trang Client không thể tự đặt giá hoặc gửi `total` để mua rẻ hơn.

## Luồng 4 — Complaint hai chiều

1. Client → **Tài khoản → Trung tâm hỗ trợ** → chọn đơn đã tạo và gửi yêu cầu.
2. Admin → **Khiếu nại**: nhìn thấy yêu cầu mới, phản hồi khách hàng.
3. Tạo một ghi chú nội bộ bằng Admin (checkbox Internal). Client không được thấy ghi chú nội bộ đó.
4. Client gửi tiếp tin nhắn, Admin nhìn thấy và có thể xử lý trạng thái.

## Luồng 5 — Đánh giá, kiểm duyệt và khóa Buyer

1. Buyer đã thanh toán đánh giá sách theo sao (không thể đánh giá sách chưa mua).
2. Từ Client báo cáo một nội dung → Admin → **Báo cáo nội dung** xử lý.
3. Nếu ẩn review, Client sẽ không còn thấy review đó.
4. Admin → **Tài khoản** khóa Buyer bằng lý do hợp lệ: token của Buyer cũ bị thu hồi; gọi `/auth/me` trả 401.

## Chạy test API tự động

```bash
npm test
```

Lưu ý: Test tích hợp tạo DB và thư mục PDF tạm riêng, **không làm thay đổi database lab của bạn**. Bộ test hiện bao gồm thực thi thật các API thay vì chỉ so sánh mock.

## Kiểm thử chưa có trong gói

- Test giao diện tự động hoàn chỉnh bằng trình duyệt.
- Cổng thanh toán thực / webhook ký / hoàn tiền.
- Pipeline antivirus cho PDF và kiểm tra bản quyền nội dung.
- Email verification, password reset, 2FA, hệ thống đồng bộ đa node.

Đây là những bước cần làm trước khi đưa hệ thống lên môi trường công khai.


## Kiểm thử chatbot Client

Chạy `npm.cmd test` trên PowerShell hoặc `npm test` trên macOS/Linux. Có các bài kiểm tra chuyên dụng `backend/tests/chatbot.test.js`:

1. Hỏi giá “Mắt biếc”, tra cứu sách giảm giá, câu hỏi tiếp theo “Giá bao nhiêu?” và FAQ khách hàng.
2. Hỏi chủ đề khác như thời tiết, viết code, bóng đá, yêu cầu tiết lộ API key: xác thực từ chối, không đưa sách và không gọi AI.
3. Admin tạo sách mới còn nháp → không thấy trong chatbot; xuất bản → thấy sách; thay đổi giá → chatbot cập nhật.
4. Bài tri thức có `PUBLISHED` nhưng `clientVisible:false` → không hiển thị; chỉ công khai khi Manager bật; DRAFT vẫn bị chặn.
5. Stub OpenAI local (không tốn token): chỉ nhận JSON chọn ID công khai, ID lạ bị bỏ, câu trả lời tùy ý từ LLM không được đưa ra Client, AI lỗi thì dự phòng tra cứu danh mục.
6. Frontend vẫn tải được HTML/CSS/JS mới, các bài kiểm tra Admin/Client cũ vẫn chạy.

**Kiểm thử giao diện thủ công:** bật server → Client `http://localhost:3000/` → nhấn **Hỏi trợ lý sách** dưới góc phải → thử câu hỏi gợi ý, hỏi tên sách, nhấn thẻ sách và kiểm tra chuyển đúng chi tiết. Thu nhỏ cửa sổ xuống khoảng 390px để kiểm tra responsive. Trên Admin, kiểm tra trang Tri thức AI và đánh dấu FAQ công khai. Không bao giờ nhập API key vào trình duyệt hoặc trang Knowledge.

## Luồng 7 — Upload, thay và xóa ảnh bìa

1. Admin Manager/Employee → Sách & PDF → nhấn một sách đang bán → **Tải ảnh bìa**; chọn JPG/PNG/WebP dưới 5MB → kiểm tra ảnh trong ô xem trước → **Lưu sách**.
2. Admin: danh sách hiện thumbnail; mở lại chi tiết có thể xem ảnh đã lưu trên backend, kể cả với sách DRAFT.
3. Client: tải lại trang chủ, danh mục, chi tiết sách, combo và chatbot → bìa sách phải khớp. Sách không có ảnh vẫn dùng CSS.
4. Thay ảnh bằng file khác và Lưu → Client tải lại thấy ảnh mới; không hiện ảnh cũ.
5. Sửa sách → **Xóa ảnh** → **Lưu sách** → Client quay về bìa mặc định.
6. Dùng Buyer token gọi `POST/DELETE/GET /admin/books/:id/cover` → 403. Sách DRAFT/HIDDEN/SUSPENDED truy xuất `GET /shop/books/:id/cover` phải trả 404. Khi upload phiên bản sách cũ (If-Match không khớp) trả 409; SVG, file giả ảnh và ảnh >5MB bị từ chối.

Test tự động: `backend/tests/book-covers.test.js` (6/6 bộ kiểm thử hiện có: cover, chatbot 3 ca, Client render, Admin↔Client). Test tạo DB và folder file **tạm** tách biệt với dữ liệu cá nhân.

## Nâng cấp mua tặng + PDF Lorem Ipsum + AI (test tự động + test tay)

Từ `webBookStore-main`, chạy `npm.cmd test` (Windows) hoặc `npm test`. Bộ `backend/tests/gifts-reader-ai.test.js` kiểm tra:

1. Người chưa mua bị chặn truy cập PDF (401/403), người mua thường thanh toán demo và nhận PDF **thật** dạng binary sinh tự động, đúng 7 trang (có xref), có dấu `Lorem ipsum` và cảnh báo nội dung demo; `view=1` trả inline, tải file trả attachment.
2. Đơn tặng không thể gắn email của người mua; không được mua số lượng > 1 cho một người nhận; sau khi trả tiền đơn có mã `GFT` hợp lệ; mã không hiện trước thanh toán.
3. Người mua **không** được quyền đọc file của đơn tặng; người khác email và mã sai bị từ chối; người nhận đăng ký đúng email, claim đúng mã -> đọc được PDF; claim lại của chính họ không tạo trùng quyền.
4. Admin nhìn thấy đơn tặng nhưng không có mã sử dụng được; người nhận không xem hóa đơn của người mua; giữ quyền của các đơn mua thường.
5. Chatbot trả lời chọn sách dưới 100k, so sánh hai cuốn theo dữ liệu public, hỏi cuốn thứ 2, quy trình nhận quà và đọc Lorem Ipsum; chặn hỏi thời tiết; lịch sử có role giả mạo bị từ chối.

**Test tay trên Windows:**

- Login `an@mail.test` → `#/book/1` → **🎁 Mua tặng** → nhập email người nhận chưa có tài khoản → tạo đơn → thanh toán demo → copy mã GFT.
- Đăng xuất, tạo Buyer bằng đúng email đó → vào `#/gifts` nhập mã → mở `#/library` → **Đọc ngay** → trình duyệt hiển thị PDF 7 trang. Kiểm tra luôn nút **Tải PDF**.
- Trên cùng trình duyệt, thử xem PDF sách chưa mua (cần có lỗi); kiểm tra Admin thấy đơn tặng tại **Orders**.
- Mở chatbot thử: `Gợi ý sách dưới 100k`, `So sánh hai cuốn này` (sau khi có hai thẻ sách), `Cuốn thứ 2 giá bao nhiêu?`, `Cách đọc PDF đã mua?`, `Mua tặng sách thế nào?`, `Thời tiết hôm nay?`.
- Kiểm tra desktop/mobile và Chrome/Edge có trình xem PDF mặc định được bật; trình xem có thể không hoạt động nếu trình duyệt chặn nội dung PDF hoặc CSP bị cấu hình lại bởi reverse proxy.

**Giới hạn kiểm thử:** Không test thanh toán thật, gửi email, xác minh sở hữu địa chỉ email hoặc uptime 24/7. Đây là hệ thống demo, không dùng mã thanh toán demo ngoài môi trường lab.
