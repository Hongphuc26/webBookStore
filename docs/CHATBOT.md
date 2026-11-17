# Chatbot tư vấn sách WebBookStore

## Phạm vi và kiến trúc

Client `frontend/client/js/chatbot.js` → `POST /api/v1/shop/chat` → Backend `backend/src/clientChat.js` → `publicCatalogForChat()` trong `shopRoutes.js` → cùng DB JSON với Admin.

Hai lớp bảo vệ phạm vi: (1) Bộ lọc chủ đề chạy trên Backend từ chối câu hỏi ngoài sách/dịch vụ mua ebook trước khi gọi AI; (2) Khi có `OPENAI_API_KEY`, AI chỉ phân loại và chọn ID sách/FAQ từ dữ liệu công khai. Backend luôn tự dựng câu trả lời từ giá, mô tả, tác giả, thể loại, bài hướng dẫn được Manager công bố; không phát câu trả lời tự do hoặc links bên ngoài do model tự tạo. Các lớp này giảm thiểu chứ không bảo đảm 100% nhận diện mọi cách diễn đạt và prompt-injection; cần tiếp tục kiểm thử khi mở Internet.

## Phạm vi trả lời

**Cho phép:** Tên, tác giả, thể loại, giá bán và giảm giá sách đang được phục vụ, gợi ý theo ngân sách/sở thích, so sánh hai sách, gợi ý lời nhắn tặng sách, hỏi nối tiếp cuốn thứ 2, hướng dẫn mua tặng/nhận quà, đọc PDF Lorem Ipsum, đơn hàng và yêu cầu hỗ trợ từ bài kiến thức công khai.

**Không cho phép:** Thời tiết, tin tức, viết code/bài tập, các bài viết không liên quan, lộ tài liệu quản trị, tiết lộ API key, hỏi sách nháp/ngưng bán như thể còn phục vụ. Với câu ngoài phạm vi, chatbot lịch sự từ chối và đưa hướng hỏi về danh mục.

## Cấu hình AI trên máy local

PowerShell sau khi `cd webBookStore-main`:

```powershell
$env:OPENAI_API_KEY="API_KEY_CUA_BAN"
$env:OPENAI_MODEL="gpt-4o-mini"
npm.cmd start
```

Không có key thì chatbot chuyển sang chế độ **catalog** (tra cứu/luật cục bộ, không phải AI). Có key thì chế độ **ai** được dùng cho ý định và đề xuất (phát sinh token phí). Nếu API AI không kết nối được, vẫn trả lời bằng danh mục công khai; không tiết lộ key. Khi dùng AI, Backend chuyển thêm phần lịch sử ngắn của chat sang dịch vụ AI để hiểu câu hỏi nối tiếp. Dữ liệu trong lịch sử được coi là input không tin cậy, tuyệt đối không chứa mật khẩu hoặc dữ liệu cá nhân nhạy cảm.

Kiểm tra `GET http://localhost:3000/api/v1/shop/chat/status`. API Chatbot không yêu cầu đăng nhập Buyer, nên không gửi thông tin riêng của đơn hàng, lịch sử mua hoặc dữ liệu tài khoản sang model.

## Cập nhật nội dung từ Admin

Vào Admin bằng tài khoản Manager → `Tri thức cho AI Admin & Client` → tạo/sửa bài → `PUBLISHED` + `Cho phép Client xem nội dung này`. Chỉ sử dụng cho hướng dẫn dành cho khách. Bài cũ mặc định chỉ dành cho Admin, trạng thái DRAFT không bao giờ hiển thị trên Client. Cấu hình Admin được lưu chung Backend và có hiệu lực ngay sau khi lưu.

## Chạy liên tục và production

Để trực 24/7, Node backend phải chạy trên VPS/server/Docker **luôn bật**, Internet tới dịch vụ mô hình phải ổn định nếu dùng AI bên ngoài, và cần reverse proxy HTTPS, rate limit ở proxy, log/giám sát, chính sách bảo mật, kiểm soát chi phí token và lưu DB production. `npm.cmd start` trên laptop không thể phục vụ khi máy ngủ/tắt. Không cam kết SLA trong bản lab.

## Quyền riêng tư

UI chỉ gửi **tối đa 6 tin nhắn gần nhất** (tối đa 400 ký tự mỗi tin) cùng tối đa 3 ID sách công khai của lượt trước để mô hình có ngữ cảnh; không lưu lịch sử chat trong DB. Nhấn nút ↺ trong chatbot để xóa nội dung trong tab. Lịch sử gửi lên Backend được kiểm tra định dạng và độ dài. Khi AI được bật, câu hỏi và dữ liệu catalog/FAQ công khai liên quan được gửi tới endpoint AI phía Backend. Không nhập dữ liệu nhạy cảm, thông tin thẻ hoặc mật khẩu vào hội thoại.
