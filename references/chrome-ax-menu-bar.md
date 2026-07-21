# Chrome AX tree menu bar takeover

## Vấn đề
Khi macOS menu bar được kích hoạt (ví dụ click vào thanh menu Chrome/Apple), toàn bộ AX tree của Chrome bị chiếm bởi `AXMenuBar` elements. Với `max_elements=5000`, không đủ node để reach tới web content + Gemini panel.

## Dấu hiệu
- get_window_state trả về `AXApplication "Chrome"` → `AXMenuBar` lặp vô hạn
- Không tìm thấy `AXWebArea`, `AXButton "Gửi"`, `AXButton "Sao chép"`
- Window title không còn "Bạn đang chia sẻ thẻ này với Gemini"

## Fix
1. Nhấn Escape 3-4 lần để dismiss menu.
2. Click vào page body (x~500, y~500 trong cửa sổ).
3. Thử lại get_window_state.
4. Nếu vẫn kẹt: navigate bằng CDP thay vì address bar:
   ```javascript
   page(action="execute_javascript", javascript='window.location.href="URL"')
   ```
5. Nếu vẫn kẹt: bring_to_front (focus lại Chrome), click, Escape, thử lại.
6. **Prevention**: Tránh click vào region y<80 (vùng menu bar) khi điều khiển Chrome.

## Phương pháp thay thế khi AX tree không hoạt động
- Dùng `page(action="get_text", ...)` để đọc nội dung web page.
- Dùng `page(action="execute_javascript", ...)` để tương tác với page.
- Dùng `web_extract(urls=[...])` để đọc nội dung từ bên ngoài.
- Dùng `list_windows` để kiểm tra window tồn tại và title.
