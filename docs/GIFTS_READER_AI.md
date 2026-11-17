# Mua tặng, nhận quà, đọc PDF demo, Chatbot AI — WebBookStore

## 1. Nâng cấp và chạy

1. Tắt server đang chạy (`Ctrl+C`). Sao lưu bản hiện tại `backend/data/db.json` và **toàn bộ** `backend/storage/`.
2. Giải nén ZIP phiên bản mới. Giữ thư mục gốc `webBookStore-main`. Chép DB và thư mục lưu ảnh/PDF đã backup vào đúng đường dẫn trong phiên bản mới; không xóa/reset dữ liệu.
3. Mở Windows PowerShell ở `webBookStore-main`, chạy `npm.cmd start`. Không cần `npm install`.
4. Client `http://localhost:3000/`; Admin `http://localhost:3000/admin/`; thử API `http://localhost:3000/health`.
5. Nếu cần AI model thật: trước khi start, chạy `$env:OPENAI_API_KEY="KEY_CUA_BAN"` và `$env:OPENAI_MODEL="gpt-4o-mini"`. Tuyệt đối không commit key hoặc đưa vào HTML/JS Client. API token có thể phát sinh phí riêng.

## 2. Mua sách cho bạn bè

**Người gửi:** Login Buyer → Khám phá sách → trang chi tiết → nút **🎁 Mua tặng**. Bạn cũng có thể vào Giỏ hàng và bật **Đây là đơn quà tặng**. Mỗi dòng sản phẩm trong giỏ tặng cần số lượng **1**. Nhập **email người nhận** (bắt buộc), tên và lời nhắn (tùy chọn) → **Tạo đơn hàng** → **Xác nhận thanh toán demo** (không thu tiền). Vào **Đơn hàng** vừa tạo → copy mã `GFT-...`. **Gửi mã riêng cho người nhận** qua kênh bạn tự chọn. Hệ thống này không tự gửi email.

**Người nhận:** Mở Client, đăng ký hoặc đăng nhập bằng đúng email người gửi đã điền → **Quà tặng** (hoặc `http://localhost:3000/#/gifts`) → nhập mã `GFT-...` → **Xác nhận nhận quà**. Quà sẽ hiện trong **Quà tặng của tôi** và sách nằm trong **Thư viện của tôi**. Người nhận không trả tiền lần nữa.

**An toàn:** Mã chỉ xuất hiện cho người mua khi đơn đã `PAID`; token được tính bằng HMAC trên khóa riêng lưu trong DB, không nằm trong source/frontend và không lưu plaintext của mã. Người nhận phải có email khớp; người mua không được quyền đọc bản quà đã mua. Người gửi có thể xem lại mã trong đơn sau khi reload. Nếu nhập sai nhiều lần sẽ bị giới hạn. Không hỗ trợ chuyển chủ quyền sau nhận trong bản lab. **Chưa xác minh email**, vì vậy email không phải bằng chứng mạnh của danh tính trong môi trường production.

## 3. Đọc PDF trong trang web

Sau khi đơn thường `PAID` hoặc bạn đã nhận quà: **Thư viện của tôi** → bên dưới bìa sách chọn **📖 Đọc ngay**. Trình xem PDF được nhúng trong Client (`#/read/<id>`); trang còn có nút **Tải PDF**. File trả từ Backend có kiểm tra token và quyền đọc trong database, không dùng link static công khai.

- **Chưa có file PDF thật từ Admin:** `backend/src/demoPdf.js` dựng một tệp PDF 7 trang có `Lorem ipsum` và cảnh báo `NOI DUNG GIA LAP`. Tựa sách trên trang đọc lấy từ catalog; bên trong PDF demo tên tiếng Việt được giản lược dấu do generator dùng font PDF mặc định, tránh phải phân phối file font. Nội dung **không phải** bản dịch/trích đoạn cuốn sách ngoài đời.
- **Admin đã upload PDF hợp lệ:** hệ thống sẽ ưu tiên tệp này. Chỉ được upload nội dung mà bạn có quyền phân phối.
- **Chưa mua/nhận quà:** API `GET /api/v1/shop/library/<id>/pdf` trả 401/403. Sách ngừng phục vụ cũng không đọc được.
- Nếu iframe PDF trắng: kiểm tra Chrome/Edge có bật PDF viewer, thử **Tải PDF** rồi mở bằng ứng dụng đọc PDF, và kiểm tra Console/CSP nếu chạy sau reverse proxy.

## 4. Trợ lý AI thông minh hơn và đúng phạm vi

Chatbot ở góc dưới Client. Các câu hỏi mẫu:

- `Gợi ý sách dưới 100k` — chỉ trả những sách đang bán, đúng giá từ Backend.
- `Tìm sách văn học để tặng bạn mới đi làm` — tư vấn sách theo danh mục và ngân sách có thật.
- `So sánh hai cuốn này` — dùng các thẻ sách gần nhất để so sánh giá, thể loại, tác giả và mô tả công khai.
- `Cuốn thứ 2 giá bao nhiêu?` — hiểu được từ ngữ cảnh tối đa 3 ID sách đang gợi ý.
- `Mua tặng sách thế nào?`, `Cách đọc PDF đã mua?`, `Gợi ý lời nhắn tặng sách` — hướng dẫn tính năng mới.
- `Viết code Python`, `Dự báo thời tiết` — ngoài phạm vi, từ chối.

**Với API key:** AI xử lý ý định, hiểu lịch sử 6 tin nhắn gần đây, chọn ID sản phẩm hoặc bài FAQ đã công khai; Backend tự kiểm tra lại danh sách sách bán, giá/sale và dựng lời đáp. Không chuyển thông tin tài khoản, đơn hàng hay khóa API sang LLM. Không có key hoặc AI timeout thì dùng bộ luật cục bộ, không tự nhận đang chạy AI.

**Giới hạn:** AI không thể đảm bảo tuyệt đối chỉ trả lời đúng chủ đề và có thể không hiểu những cách diễn đạt chưa được kiểm thử. Không hỏi AI thông tin đơn hàng cá nhân vì chatbot công khai không được quyền truy cập. Tránh nhập thông tin nhạy cảm trong chat.

## 5. Kiểm thử và API

Chạy `npm.cmd test`: các bộ test tự động có sẵn và `backend/tests/gifts-reader-ai.test.js` sẽ kiểm tra luồng mua tặng, từ chối đọc không có quyền, PDF binary 7 trang, cùng câu hỏi chatbot. Contract chi tiết ở `docs/API.md`; hướng dẫn thử thủ công ở `docs/TESTING.md`.

**Triển khai thực tế cần bổ sung:** PostgreSQL/MySQL với transaction, xác minh email, thanh toán bằng cổng thật có webhook ký số, giới hạn dùng/cost AI theo tài khoản, HTTPS, CSRF/CORS phù hợp deployment, backup/restore secret và file PDF, giám sát server 24/7. Bản ZIP này chỉ dành cho demo và học tập.
