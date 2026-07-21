# Session learnings — 2026-07-21

## Claude Code CLI là luồng chính (thay thế Claude.app)

**Vấn đề cũ**: Claude.app không đọc được local file, Cmd+V bị webview chặn, type_text timeout.
**Fix**: Dùng Claude Code CLI (`claude -p`) để đọc file + tạo PHP patch trực tiếp.

**Luồng mới**:
1. Gemini trả biên bản → trích xuất → `/tmp/bienban_XXXX.txt`
2. `claude -p 'Tạo inc/patch_XXXX.php. Đọc bienban_XXXX.txt. Đọc inc/patch_1181.php (mẫu).'`
3. Claude Code tự đọc file, tạo patch, kiểm tra toán học
4. Commit + push GitHub

**Path issue**: `~/.local/bin` không có trong PATH mặc định của Hermes terminal. Cần `export PATH="$HOME/.local/bin:$PATH"` trước mọi lệnh `claude`.

**Auth issue**: `claude --bare` cần `ANTHROPIC_API_KEY` (không set). Dùng `claude auth login` (OAuth browser) rồi `claude -p` không `--bare`.

**Auth login với PTY**: `claude auth login --console` (không PTY) → timeout 124. Dùng `pty=true` trong terminal() để chạy TUI login flow thành công.

**Timeout**: CLI foreground mode timeout 300s. Task lớn (25 turns+) → split thành nhiều lần chạy, hoặc dùng Web Remote.

## Claude Code Web Remote — đã kiểm chứng hoạt động ổn định

- `https://claude.ai/code` — dashboard quản lý Claude Code sessions  
- Session "dethimontoan.net" hiển thị với trạng thái Idle/Error/Running  
- Có thể type instruction trực tiếp vào Prompt và Send  
- Dùng khi CLI bị auth issue  
- **LUỒNG CHÍNH MỚI** (ưu tiên hơn CLI):  
  1. Mở `claude.ai/code` trong Chrome  
  2. Click session "dethimontoan.net" trong sidebar  
  3. Type instruction vào Prompt → Click Send  
  4. Claude xử lý real-time (có logs), tự động push lên GitHub  
  5. Đợi session về "Idle" → xong  

**Important: quản lý tab cẩn thận**
- Để Claude Code tab yên — KHÔNG navigate nó đi
- Nếu cần navigate exam URL, dùng TAB khác (click radio button trong AX tree)
- Dễ lẫn tab khi dùng page() vì page() navigate TAB HIỆN TẠI — nếu đang ở claude.ai/code, nó navigate đi mất session

## Merge conflict resolution — CẠM BẪY sed

**Vấn đề**: Remote (GitHub) có changes từ user fix cPanel → push bị reject.
**Giải pháp**: `git pull --rebase origin main`

**Tuyệt đối KHÔNG dùng sed để resolve conflict**:
- `sed -i '' '/^<<<<<<< HEAD/,/^>>>>>>>/d'` KHÔNG hoạt động đúng
- Nó CHỈ xóa markers (<<<<, ====, >>>>) nhưng GIỮ NGUYÊN nội dung giữa chúng
- Kết quả: HEAD + THEIR version đều còn → duplicate function → 500

**Cách resolve đúng (thủ công)**:
1. Đọc file conflict → xác định HEAD vs THEIR block
2. Chọn GIỮ 1 phiên bản (thường HEAD)
3. Xóa hoàn toàn bên kia (bằng patch tool hoặc Python)
4. Kiểm tra syntax: không 2 function cùng tên, không brace mồ côi
5. Commit → push → user deploy

**Triệu chứng sed sai → PHP error:**
- `Parse error: syntax error, unexpected token "*"` → orphaned `*` từ comment
- `Unclosed '{' on line 267` → brace mồ côi
- `Cannot redeclare ttp_patch_1181_r3_v1()` → duplicate function

## Claude Code r2 có thể sai lệch so với đề xuất Gemini

- Claude Code có thể tự sửa khác biên bản (VD: sa4 thành `(x+2)^2-x(x-3)=18` thay vì `(2x-3)/2-(x+1)/3=1`)
- **LUÔN so sánh output Claude Code với raw_bienban_XXXX.txt trước khi push**
- Nếu sai → update: "Đọc raw_bienban_XXXX.txt → cập nhật theo đúng đề xuất Gemini"

## Kiểm tra site sau fix

- Dùng `web_extract` với URL đề → HTTP 200 = site ổn
- Nếu web_extract load được nhưng nội dung chưa đổi → patch chưa chạy (server chưa deploy)
- Cần user deploy: `git pull` trên server hoặc cPanel

## Gemini "Hỏi Gemini" button

- Nằm trong AXWebArea (page content), KHÔNG phải toolbar
- Index ~1348-1500 trong get_window_state
- Profile Vô danh VẪN có
- Có thể có 2 bản copy — click cái đầu tiên

## Claude Code tạo patch workflow

1. Prompt cần chỉ rõ: đọc biên bản, đọc patch mẫu, xem đề gốc
2. Nếu Claude Code hỏi chi tiết (barem, q+ans) → cung cấp từ web_extract
3. Claude Code tự verify toán học trước khi ghi file
4. KHÔNG cần php -l (không có PHP CLI local)

## r1 vs r2 pattern

- r1 = fix đầu tiên (priority 40)
- r2 = fix lần 2 chồng lên r1 (priority 41)  
- Mỗi revision: hàm + option DUY NHẤT, idempotent gate
- r2 chỉ đụng các câu CHƯA được r1 sửa
