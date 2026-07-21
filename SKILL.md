---
name: dethimontoan-review-loop
description: 'Vòng lặp thẩm định đề thi dethimontoan.net: Gemini → trích xuất biên bản → Claude Code (Web Remote/CLI) tạo PHP patch → push GitHub → reload → Gemini lại → đến khi ĐẠT → chuyển đề kế tiếp.'
---

# Vòng lặp thẩm định đề thi dethimontoan.net

## Mục tiêu
Chrome → Gemini skill → trích xuất biên bản → commit raw lên GitHub → **Claude Code Web Remote (claude.ai/code)** hoặc CLI → tạo PHP patch → push GitHub → deploy → reload Chrome → Gemini lại → ĐẠT → chuyển đề mới (Lớp 9→10→11→12).

## Danh sách đề (lấy URLs)
```bash
# Lớp 9: web_extract('https://dethimontoan.net/kho-de-thi/lop-9/')
# Lớp 10: web_extract('https://dethimontoan.net/kho-de-thi/lop-10/')
# Lớp 11-12: thay lop-N/
# Parse URLs từ danh sách, chạy lần lượt từng mã đề MIỄN PHÍ trước, trả PHÍ sau
# Lớp 9 miễn phí: 1182→1183→1184→1185 (theo thứ tự)
```

## Luồng chính
**Claude Code Web Remote** (https://claude.ai/code) — ổn định hơn CLI.
**FALLBACK**: Claude Code CLI (`claude -p`) — cần PATH.
**CUỐI CÙNG**: Claude.app desktop.

## 6 Bước

### Bước 1 - Navigate Chrome đến đề
- **Cách 1** (ưu tiên): `Cmd+L` → type URL → Return (page() hay bị chặn khi Gemini sharing)
- **Cách 2**: `page(action="execute_javascript", javascript="window.location.href='URL'")` — navigate TAB HIỆN TẠI
- Đợi ~5s. Verify title có mã đề.
- **LUÔN kiểm tra tab trước khi navigate**: nếu đang ở claude.ai/code, navigate mất session

### Bước 2 - Chạy skill Gemini

1. **Mở Hỏi Gemini panel**:
   - Cách 1: Click extensions (Tiện ích) → tìm `AXButton "Hỏi Gemini"` trong popup → click
   - Cách 2: Pixel click (1460, 5) — nếu extension được pin
   - CHỈ 1 LẦN (toggle). Click 2 lần = tắt.
   - Verify: `get_window_state` → window title có "Bạn đang chia sẻ thẻ này với Gemini"

2. **Bắt đầu cuộc trò chuyện mới** (nếu có lịch sử cũ):
   - `get_window_state(max_elements=5000)` → tìm `AXButton "Bắt đầu cuộc trò chuyện mới"`
   - Click nó (AXPress).

3. **Gọi skill**:
   - `press_key("f6", FG)` focus input
   - `type_text("/", FG)` — verified:true = vào đúng ô
   - `get_window_state` → tìm `AXMenuItem "Thẩm định đề thi - dethimontoan.net"`
   - Click skill → Click `AXButton "Gửi"`
   - **LUÔN get fresh state trước mỗi click** — element index thay đổi mỗi snapshot

4. **Chờ kết quả — Poll 15s** (không sleep 120):
   - `get_window_state` mỗi 15s, tìm "BIÊN BẢN THẨM ĐỊNH" + "Gemini là một AI" trong result
   - Tối đa 20 lần (5 phút). Nếu quá → reload tab + chạy lại.
   ```python
   from hermes_tools import terminal
   import time
   for i in range(20):
       time.sleep(15)
       r = terminal("grep -c 'BIÊN BẢN THẨM ĐỊNH.*1182' /tmp/result.txt")
       if 'done': break
   ```

### Bước 3 - Trích xuất biên bản (NGUYÊN VĂN)

**Phương pháp 1 (khuyến nghị) — từ structured elements**:
Dùng kết quả của `get_window_state(max_elements=2000, query="BIÊN BẢN|Mã đề")`. Biên bản nằm trong `elements[].label` của các node `AXStaticText` dưới Gemini panel. Ghép các label kề nhau từ "BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ:" đến "Gemini là một AI".

**Phương pháp 2 — từ file dump (fallback)**:
- **NGUYÊN VĂN, KHÔNG RÚT GỌN**
- KHÔNG dùng nút "Sao chép" (AXPress không trigger clipboard)

### Bước 4 - Claude Code tạo PHP patch

**PHƯƠNG PHÁP 1 — Claude Code Web Remote (ƯU TIÊN)**:
1. Trích xuất biên bản → `/tmp/bienban_XXXX.txt` (NGUYÊN VĂN)
2. Copy vào repo: `cp /tmp/bienban_XXXX.txt /tmp/dethimontoan.net/raw_bienban_XXXX.txt`
3. Commit lên GitHub: `git add && git commit -m "raw bien ban XXXX" && git push`
4. Mở `https://claude.ai/code` trong Chrome
5. Click session "dethimontoan.net" trong sidebar
6. `type_text(instruction, FG)` vào Prompt → `press_key(return)` — KHÔNG FG
7. Sidebar hiện "Running dethimontoan.net"
8. **Tự động poll 15s** kiểm tra sidebar còn Running không:
   - `get_window_state` query "Running dethimontoan" — nếu không còn → done
9. Kiểm tra commit mới trên GitHub sau khi done

**Yêu cầu mẫu cho Claude Code Web**:
```
Đọc raw_bienban_XXXX.txt (biên bản Gemini) và inc/patch_XXXX.php (patch hiện tại).
Cập nhật patch_XXXX.php để khớp chính xác đề xuất Gemini.
Giữ NGUYÊN các fix cũ (r1, r2...). Bump option nếu cần. Commit push.
```

**PHƯƠNG PHÁP 2 — Claude Code CLI (FALLBACK)**:
```bash
export PATH="$HOME/.local/bin:$PATH"
cd /tmp/dethimontoan.net
claude auth status --text || claude auth login
claude -p '...' --allowedTools "Read,Write,Edit,Execute" --max-turns 40
```
⚠️ CLI timeout 300s (mặc định). Cần `pty=true` trong terminal() vì Claude CLI cần PTY. Dùng `--allowedTools "Read,Write,Edit,Execute"` (không "WebFetch" — WebFetch hay bị lỗi permission). Nếu session chặn git write, tự tay `git add/commit/push` sau.

**⚠️ Quản lý tab khi dùng Web Remote**:
- Tab claude.ai/code KHÔNG navigate đi nơi khác
- Khi cần navigate exam URL: dùng TAB KHÁC hoặc Cmd+L
- page() navigate TAB HIỆN TẠI → chỉ dùng khi tab đang ở exam

**Commit & push**:
```bash
cd /tmp/dethimontoan.net
git add inc/patch_XXXX.php
git commit -m "patch_XXXX rN: fix ..."
# Remote có thể có changes (user fix cPanel) → pull rebase
git pull --rebase origin main && git push origin main
```
- **KHÔNG dùng sed để xóa conflict markers** — gây lỗi PHP parse error
- Cách resolve đúng: đọc file → giữ 1 phiên bản (HEAD hoặc THEIR)

### Bước 4.2 - Tự động theo dõi Claude Code (poll 15s)
```python
from hermes_tools import [terminal, mcp__cua_driver__get_window_state]
import time
for i in range(40):  # Tối đa 10 phút
    time.sleep(15)
    r = get_window_state(pid=..., window_id=..., max_elements=100, query="Running dethimontoan")
    if 'Running' not in r['result']: break  # Done
# Kiểm tra commit mới
terminal("cd /tmp/dethimontoan.net && git pull && git log --oneline -3")
```

### Bước 5 - Reload → Gemini lại
- Nếu đang ở claude.ai/code: chuyển tab exam (dùng tab khác), reload Cmd+R
- Nếu đã ở exam tab: Cmd+R reload
- Đợi ~5s → Bước 2 (bỏ qua "Bắt đầu cuộc trò chuyện mới" nếu Gemini đã share)

### Bước 6 - Chuyển đề tiếp theo
- **"TOÀN BỘ ĐỀ [MÃ] ĐÃ ĐẠT CHUẨN..."** → ✅ Đề này OK.
  - **KHÔNG dừng.** Chuyển sang đề kế tiếp trong danh sách.
  - Quay lại **Bước 1** (navigate URL đề mới).
- **Còn bất kỳ LỖI nào** → quay lại Bước 3 (extract biên bản) → Bước 4 (Claude Code fix).
  - **Cứ lặp fix→check cho đến khi Gemini báo ĐẠT mới chuyển đề.**
  - Không bỏ qua lỗi nào.
- **CẢNH BÁO TC9 (barem)**: Nếu Gemini báo LỖI TC9 nhưng barem đã thêm → kiểm tra deploy. Chưa deploy = Gemini thấy đề cũ.

## Pitfalls

### Gemini panel không mở
- Dùng Chrome bar Gemini: tìm `AXMenuBarItem [help="Bật/tắt Gemini trong Chrome"]` trong `get_window_state` và press nó 1 lần.
- Nếu không thấy node Gemini, mới fallback extension popup `AXButton "Hỏi Gemini"` hoặc sau đó click vùng panel.
- Verify: `press_key("f6", FG)` → `type_text("/", FG)` — verified:true = panel mở + focus đúng ô.
- **Lưu ý**: browser `page()` không nhìn thấy Chrome side panel; nên dùng `get_window_state` + AX/MenuBarItem.
- KHÔNG kill Chrome (gây crash dialog)
- **Delivery mode**: Các thao tác bàn phím với Gemini (F6, Return, type_text) thường cần `delivery_mode="foreground"` vì Gemini panel là web area phụ — background key events hay bị chặn.

### Element index thay đổi
- Mỗi `get_window_state` tạo snapshot mới → index thay đổi
- LUÔN get fresh state trước mỗi click
- Dùng element_token cho độ tin cậy cao hơn

### Gemini đề xuất khác mỗi lần — dùng biên bản mới nhất
- Mỗi lần Gemini thẩm định có thể báo LỖI KHÁC nhau (cùng TC4 nhưng lỗi cũ hết, lỗi mới xuất hiện)
- **LUÔN dùng biên bản lần chạy gần nhất** làm căn cứ fix, không dùng biên bản cũ
- Sau mỗi vòng fix→check, extract biên bản MỚI (Bước 3), không tái sử dụng file cũ
- Triệu chứng: Gemini báo TC4 hết lỗi nhưng TC9 lỗi mới → fix TC9, không đụng lại TC4

### Luôn kiểm tra bằng ảnh màn hình trước khi kết luận đã gửi/thao tác thành công
- Nếu muốn biết chính xác prompt đã gửi/hiện lên hay chưa: **chụp ảnh màn hình** rồi kiểm tra trực tiếp, đừng đoán từ output `verified:true/false`.
- Nếu thấy chưa gửi/hiện đúng chỗ → gửi lại ngay bằng đúng luồng đang mở, đừng chuyển sang phương án khác khi chưa xác nhận bằng hình.
- Đây là quy tắc áp dụng cho cả Gemini, Claude Code và mọi thao tác UI có thể nhìn thấy bằng screenshot.

### Claude Code không chạy (sidebar không chuyển Running)
- Gửi instruction xong, sidebar vẫn "Idle" → lệnh chưa đến prompt
- **Fix**: reload tab claude.ai/code, click session lại, type_text FG lại, press_key return
- Nguyên nhân: press_key return KHÔNG FG hay bị chặn; type_text vào prompt sai element

### Merge conflict khi push
- Remote có changes → `git pull --rebase origin main`
- CẤM dùng sed xóa conflict markers → "Parse error: unexpected token '*'" / "Unclosed '{'"
- Cách resolve: đọc file, giữ 1 phiên bản duy nhất

### Deploy patch cần thiết — GitHub push ≠ live
- GitHub push xong, patch CHƯA có hiệu lực trên WordPress
- Webhook deploy tự động, nhưng cần kiểm tra lại
- Nếu Gemini vẫn báo LỖI giống hệt lần trước → chưa deploy
- Triệu chứng: "Câu 4 (7x=0)" vẫn xuất hiện → patch chưa chạy

### Gửi biên bản cho Claude Code — YÊU CẦU CỨNG
- **GUYỀN TẮC**: Gửi cho Claude Code PHẢI là **nội dung ĐẦY ĐỦ và CHÍNH XÁC 100%** của biên bản thẩm định Gemini. **TUYỆT ĐỐI KHÔNG rút gọn, tóm tắt, hay diễn giải lại biên bản.**
- Claude Code PHẢI fix **chính xác theo đề xuất trong biên bản**, **KHÔNG được lệch hướng fix lòng vòng** hay tự ý thay đổi yêu cầu.
- Nếu Claude Code tạo patch KHÁC đề xuất Gemini → **dừng ngay**, đọc raw_bienban, gửi lệnh update chính xác theo biên bản.
- Bug thường gặp: Claude Code chỉ apply `e` (barem), quên `q`+`ans` cho sa — bump gate option để chạy lại.
- **Fallback bắt buộc**: Nếu Claude Code Web Remote/CLI treo/timeout → **dừng retry vô hạn**, tự tạo `inc/patch_XXXX_rN.php` dựa trên `raw_bienban_XXXX.txt`, commit + push ngay.
- Tạo fallback patch theo nguyên tắc: đúng đối tượng, đúng field `q/o/c/ans/e`, guard theo nội dung mới để idempotent, không động phần đã đạt.

### Kiểm tra site sau fix
- Dùng `web_extract(URL)` để verify nội dung (nhanh hơn browser)
- Nếu load được nhưng nội dung chưa đổi → patch chưa chạy

### WordPress PHP error sau push
- Symptom: "Parse error" / "Unclosed '{'" / "Unexpected '*'"
- Do merge conflict resolve sai
- Fix: commit fix → push → user deploy

### Biên bản trùng (duplicate) trong AX tree
- Nội dung page xuất hiện 2 lần (panel + page)
- Dùng dấu hiệu "Gemini là một AI" làm điểm dừng
- Dùng `structuredContent.elements` để truy cập chính xác

### Chrome window bị stale
- `list_windows` → lấy window_id mới nếu cũ không hoạt động
- `bring_to_front` để focus window

## Luật bất biến
- **GUỬI NGUYÊN VĂN, KHÔNG RÚT GỌN DÙ 1 TỪ**
- **KHÔNG hỏi user** — tự retry/reset, KHÔNG hỏi "có muốn tiếp tục không"
- **Reload tab trước mỗi lần chạy Gemini**
- **Luôn get fresh state trước click**
- **Tự động poll 15s — không sleep 120s**
- **Sau Claude Code: verify output khớp biên bản gốc**
- **ĐẠT → chuyển đề kế tiếp, không dừng**
