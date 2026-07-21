# Known pitfalls (từ session thực tế)

## URL navigation
- **Vấn đề**: URL bị nối khi type_text không clear address bar trước.
- **Fix**: Luôn Cmd+L -> Cmd+A -> Delete trước khi type URL mới.

## Sao chép button
- **Vấn đề**: Click AXButton "Sao chép" không copy vào clipboard (clipboard rỗng sau click).
- **Fix**: Thử click element thứ 2 ("Sao chép") hoặc dùng pixel click tại tọa độ button (lấy frame từ get_window_state). Nếu vẫn không được, trích xuất text từ AX tree và copy bằng `pbcopy`.

## Paste vào Claude.app
- **Vấn đề**: Cmd+V không đổ vào Claude.app webview (Electron chặn paste event).
- **Fix**: `type_text` chia ~700 ký tự/lần, dùng ` | ` làm separator thay newline.
- **Lưu ý**: `set_value` KHÔNG hoạt động (WebKit bỏ qua AXValue write).
- **Nội dung**: PHẢI là nguyên văn từ Gemini, không rút gọn.

## Claude session sai
- **Vấn đề**: Gửi biên bản vào session "Áp dụng prompt cho hai website toán" thay vì "dethimontoan.net".
- **Fix**: Luôn click AXButton "dethimontoan.net" ở sidebar trước khi focus Prompt. Tránh click vào session có prefix "Error".

## Claude chạy quá lâu / báo Error
- **Vấn đề**: Claude "Running dethimontoan.net" kéo dài 5-7 phút. Hoặc "Error dethimontoan.net" xuất hiện (thường do content sai hoặc gửi nhầm session).
- **Các trạng thái session**: "dethimontoan.net" (sẵn sàng), "Running dethimontoan.net" (đang xử lý), "Error dethimontoan.net" (lỗi, cần session mới), "Unread response dethimontoan.net" (có phản hồi).
- **Fix**: Nếu "Error" - click "New session" (element 8) tạo session mới, gửi lại. Nếu "Running" >5 phút - click Stop button ở pixel ~(1224,879).
- **Phòng tránh**: Luôn xác nhận click đúng session sidebar TRƯỚC KHI gửi. Claude có 2+ AXWindow -> nhiều Prompt textarea. Gửi sai Prompt -> vào session khác -> "Error".

## Chrome AX tree bị menu bar chiếm
- **Vấn đề**: macOS menu bar (Apple/Chrome/Tệp/Sửa/...) chiếm toàn bộ 5000+ AX nodes, không reach được Gemini panel.
- **Fix NHANH (session 2026-07-20)**: `killall SystemUIServer; sleep 2; osascript -e 'tell app "Google Chrome" to activate'`
- **Fix cũ**: Nhấn Escape nhiều lần để dismiss menu, click vào page body (x~500, y~500). Nếu vẫn kẹt: navigate đến URL khác bằng CDP.
- **Prevention**: Tránh click hoặc hover vùng y < 33 (menu bar area). Chỉ Chrome bị - Claude.app không bị.

## Gemini skill không kích hoạt
- **Vấn đề**: Gõ "/Thẩm định đề thi - dethimontoan.net" rồi Enter chỉ gửi tin nhắn thường, không kích hoạt skill.
- **Fix**: Gõ `/` -> đợi menu "Kỹ năng của bạn" hiện -> CLICK vào mục skill (thành chip vàng). Sau đó click "Gửi".

## Gemini web app (gemini.google.com) khác built-in sidebar
- **Vấn đề**: Gemini trên gemini.google.com KHÔNG phải sidebar Chrome. Ô nhập là div.ql-editor (contenteditable Quill), không hiện trong AX tree dưới dạng AXTextArea/AXTextField.
- **Phân biệt**: Sidebar = button "Bật/tắt Gemini trong Chrome" (AXMenuBarItem, help="Bật/tắt Gemini trong Chrome"); Web app = tab riêng với URL gemini.google.com.
- **Hậu quả type_text rơi vào address bar**: click_element(selector=".ql-editor") không focus được input -> type_text gõ vào address bar -> Chrome navigate đến URL lạ (file:///Thẩm%20định%20...). Mất nhiều lần retry.
- **Fix**: 
  - Cách 1 (sidebar, ưu tiên): click button Gemini toolbar, chat input nằm trong AX tree gốc.
  - Cách 2 (web app + CDP): launch Chrome với `cdp_debugging_port` và `additional_arguments: ["--user-data-dir=<path>"]` -> dùng insert_text/type_keystrokes (CDP path).
  - Cách 3 (pixel click): tìm tọa độ input trên window và click bằng pixel coordinates.

## Element cache hết hạn
- **Vấn đề**: "Element index 3821 not found in cache" - mỗi lần `get_window_state` tạo snapshot mới, index thay đổi.
- **Fix**: Gọi `get_window_state` với `max_elements=5000` trước mỗi lần click, dùng index từ snapshot mới nhất.

## Chrome pid thay đổi
- **Vấn đề**: Chrome pid cũ bị "process not found" khi Chrome restart.
- **Fix**: `list_windows` để tìm pid/window mới của Chrome (title chứa dethimontoan.net).

## Không dùng vision_analyze
- **Vấn đề**: deepseek/deepseek-v4-flash không hỗ trợ vision (không xem được ảnh user gửi).
- **Fix**: Dựa vào text user gửi hoặc web_extract để đọc nội dung.

## Biên bản verbatim là bắt buộc
- **Vấn đề**: User nhiều lần phê bình "nội dung biên bản chưa đúng" vì copy rút gọn thay vì nguyên văn Gemini. Đã xảy ra 8+ lần trong session 2026-07-20.
- **Fix CHÍNH THỨC (FILE-BASED)**: Trích xuất từ AX tree -> lưu /tmp/bienban_XXXX.txt. Gửi Claude 1 dòng: "Fix Mã XXXX. Đọc /tmp/bienban_XXXX.txt. Merge." Claude tự đọc file -> fix chính xác nguyên văn.
- **TẠI SAO**: type_text >1000 ký tự bị timeout. Agent có xu hướng rút gọn -> user phát hiện ngay.
- **KHÔNG BAO GIỜ**: type_text chunks rút gọn. User đã từ chối cách này nhiều lần.
- **XÁC NHẬN**: File-based approach đã hoạt động thành công (session dethimontoan.net hiện "Running" với 78 ký tự type_text).
