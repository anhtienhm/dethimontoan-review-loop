# Session learnings 2026-07-21 đề 1183

## Claude Code CLI issues
- `claude -p` treo mãi (>180s) khi đọc raw_bienban_1183.txt → fallback sang Claude Code Web Remote hoặc tự tạo patch trực tiếp
- Background `claude` interactive cần handle trust dialog (chọn `1` rồi submit), nhưng vẫn có thể treo do limit/effort

## Biên bản rõ ràng quan trọng
- Khi Gemini báo lỗi lần sau, PHẢI dùng BIÊN BẢN MỚI NHẤT làm căn cỡ fix, không dùng biên bản cũ
- Biên bản cần ghi rõ: số lần thử, các lỗi còn lại, đề xuất sửa chính xác từng câu

## Claude Code Web Remote
- CMD+L mở new tab (Omnibox popup), không mở Claude Code trực tiếp
- Cần navigate thủ công đến claude.ai/code hoặc dùng menu khác
- Khi Web Remote chạy xong, fetch không nhận đủ nội dung câu → dùng page() execute_javascript để đọc nội dung trang

## Patch PHP strategy
- Khi Claude Code không chạy được, tự tạo patch file mới `patch_XXXX_rN.php` đúng cú pháp PHP
- Guard theo field `q` để đảm bảo idempotent
- Không dùng `unset($data[$i]['fig'])` nếu không chắc chắn tồn tại field (thêm `isset` check)
- Barem sa phải đủ 4 bước rõ ràng theo chuẩn Gemini

## Biên bản cuối đề 1183
- ĐÃ ĐẠT sau khi sửa: Câu 7, 11 (tc4), Câu 1,3,5 (tc9 barem)
- Patch r2 đã push, cần deploy để có hiệu lực
