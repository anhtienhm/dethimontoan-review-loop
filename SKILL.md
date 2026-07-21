---
name: dethimontoan-review-loop
description: 'Vòng lặp thẩm định đề thi dethimontoan.net: Gemini → trích xuất biên bản → Claude Code (Web Remote/CLI) tạo PHP patch → push GitHub → reload → Gemini lại → đến khi ĐẠT → chuyển đề kế tiếp.'
---

# Vòng lặp thẩm định đề thi dethimontoan.net

## Mục tiêu
Chrome → Gemini skill → trích xuất biên bản → commit raw lên GitHub → **Claude Code Web Remote (claude.ai/code)** hoặc CLI → tạo PHP patch → push GitHub → deploy → reload Chrome → Gemini lại → ĐẠT → chuyển đề mới (Lớp 9→10→11→12).

## Danh sách đề (lấy URLs)
```bash
# DANH SÁCH CHUẨN (URL đầy đủ + trạng thái ✅/⏳): xem references/exam-list.md — cập nhật file đó khi xong mỗi đề
# Lớp mới: web_extract('https://dethimontoan.net/kho-de-thi/lop-N/') → parse URLs
# Chạy lần lượt từng mã đề MIỄN PHÍ trước, trả PHÍ sau
# Lớp 9: 1182 ✅ → 1183 🔄 (đã fix, CHỜ DEPLOY + verify lại) → 1185 → 1186 → 1187 → 1188 → 1189 (KHÔNG có mã 1184)
```

## Luồng chính
**Claude Code Web Remote** (https://claude.ai/code) — ổn định hơn CLI.
**FALLBACK**: Claude Code CLI (`claude -p`) — cần PATH; hay treo >180s khi đọc raw file → đặt timeout, treo thì quay về Web Remote.
**CUỐI CÙNG**: Claude.app desktop (KHÔNG đọc được local file — chỉ type_text chunks).
Chi tiết từng phương pháp + cách gửi biên bản: `references/paste-methods.md`.

## 6 Bước

### Bước 1 - Navigate Chrome đến đề
- **Cách 1** (ưu tiên): `Cmd+L` → type URL → Return (page() hay bị chặn khi Gemini sharing)
- **Cách 2**: `page(action="execute_javascript", javascript="window.location.href='URL'")` — navigate TAB HIỆN TẠI
- Đợi ~5s. Verify title có mã đề.
- **LUÔN kiểm tra tab trước khi navigate**: nếu đang ở claude.ai/code, navigate mất session

### Bước 2 - Chạy skill Gemini

1. **Mở Hỏi Gemini panel** — hiểu đúng trước khi thao tác:
   - TIỀN ĐỀ: chạy trên PROFILE Chrome THƯỜNG đã đăng nhập Google — KHÔNG Guest/"Vô danh" (guest làm extension từ chối synthetic toggle: 21/7 mọi hotkey/AXPress/click đều câm trên cửa sổ Vô danh, trong khi các phiên 1182 thành công đều ở profile thường). Gom về MỘT cửa sổ Chrome chứa tab đề (thu nhỏ/bỏ qua cửa sổ khác, KHÔNG kill Chrome). Hotkey/menu-item/click đánh vào CỬA SỔ ACTIVE — action và verify PHẢI cùng một window_id (list_windows lấy id mới → bring_to_front → xác nhận frontmost bằng title). Nhiều cửa sổ + verify sai target → chuỗi cách 1→4 thành toggle MỞ-ĐÓNG-MỞ-ĐÓNG (đã vấp thật 21/7 với 10 cửa sổ)
   - Sau MỖI lần toggle: CHỤP SCREENSHOT xác nhận bằng mắt rồi mới kết luận fail
   - Panel chào "Xin chào Vô danh" = CHƯA đăng nhập Google → menu `/` sẽ KHÔNG có skill "Thẩm định đề thi - dethimontoan.net" (skill gắn theo tài khoản) → phải đăng nhập đúng tài khoản TRƯỚC khi chạy tiếp
   - Panel là SIDE PANEL bên trong cửa sổ Chrome, KHÔNG phải cửa sổ riêng → `list_windows` không bao giờ thấy "cửa sổ Gemini" (đó KHÔNG phải dấu hiệu lỗi)
   - Chip ✦ "Hỏi Gemini" KHÔNG xuất hiện trong AX tree khi panel chưa mở (không thấy element ≠ không có extension) → KHÔNG kết luận blocker từ get_window_state
   - VỊ TRÍ chip (đo screenshot 21/7, cửa sổ 2000px): HÀNG TAB STRIP (cùng hàng các tab, dưới menu bar, TRÊN toolbar), sát mép phải cửa sổ — tâm ≈ `(W−85, 71)` theo cửa sổ (2000px → ~(1915, 71)); hàng toolbar ngay dưới (y≈130) là Tiện ích (W−152) · avatar (W−80) · ⋮ (W−32) — đừng click nhầm hàng
   - Icon ✦ THỨ HAI trên MENU BAR macOS (~x1357, y22) = `AXMenuBarItem [help="Bật/tắt Gemini trong Chrome"]` — chỉ dùng AXPress, KHÔNG pixel click (y<44 là menu bar → pitfall takeover)
   - Nút GHIM VÀO TOOLBAR (từ 21/7 user đã pin qua Tiện ích — layout hiện hành): nằm CÙNG HÀNG thanh địa chỉ, GIỮA icon Tiện ích và avatar profile, NGAY TRÁI avatar ~105px (đo 2000px: Tiện ích x≈1690 · Gemini ≈(1815, 32) · avatar x≈1920). Nút ghim toolbar là `AXButton "Hỏi Gemini"` CHUẨN → ƯU TIÊN click theo ELEMENT trong AX tree; anchor dự phòng: `(X_avatar − 105, Y_avatar)` — cùng hàng, KHÔNG trừ theo chiều dọc

   Thứ tự thao tác (mỗi cách CHỈ TOGGLE 1 LẦN rồi verify ngay, fail mới sang cách kế):
   - Cách 0 — KIỂM TRA ĐÃ MỞ CHƯA: `get_window_state` query "Đang chia sẻ|Gemini Chrome" → thấy = panel ĐANG mở, DỪNG (toggle nữa = tắt)
   - Cách 1 (nút đã ghim TOOLBAR — ưu tiên nhất): `get_window_state` fresh query "Hỏi Gemini" → thấy `AXButton` → click theo ELEMENT (không pixel) → verify; không thấy element → click anchor `(X_avatar − 105, Y_avatar)` từ element avatar → screenshot verify
   - Cách 1b (không cần toạ độ/AX): `bring_to_front` Chrome → `hotkey(["cmd","shift","y"], FG)` → đợi ~2s → verify
   - Cách 2: lấy AX tree mức APP/menu bar (get_window_state theo window KHÔNG thấy) → `AXMenuBarItem [help="Bật/tắt Gemini trong Chrome"]` → AXPress 1 lần → verify
   - Cách 2b — System Events qua `terminal()` (ĐƯỜNG TIÊM KHÁC driver — dùng khi mọi click/hotkey của driver câm; cần quyền Accessibility, thường đã có):
     ```bash
     # Thăm dò: nút "Hỏi Gemini" có trong AX của System Events không (get_window_state không thấy ≠ System Events không thấy)
     osascript -e 'tell application "System Events" to tell process "Google Chrome" to get name of every button of window 1'
     # Có tên chứa Gemini → click thẳng theo tên:
     osascript -e 'tell application "Google Chrome" to activate' -e 'delay 0.5' \
               -e 'tell application "System Events" to tell process "Google Chrome" to click (first button of window 1 whose name contains "Gemini")'
     # Không có nút → gõ phím tắt qua System Events:
     osascript -e 'tell application "Google Chrome" to activate' -e 'delay 0.5' \
               -e 'tell application "System Events" to keystroke "y" using {command down, shift down}'
     ```
     Lỗi "not allowed assistive access" → nhờ user cấp quyền Accessibility cho app chạy terminal (setup 1 lần)
   - Cách 3: toolbar `AXPopUpButton "Tiện ích"` (query "Tiện ích", KHÔNG query "Gemini" — tên nút không chứa chữ Gemini) → click → fresh state → `AXButton "Hỏi Gemini"` trong popup → click → Escape đóng popup → verify
   - Cách 4 (cuối): CHỤP SCREENSHOT, xác định tâm chip ✦ bằng mắt (ước lượng `(W−85, 71)`) → pixel click đúng tâm; KHÔNG dùng toạ độ hardcode cũ (1460, 5)
   - Cách 5 (đủ cách 1→4 × 2 vòng vẫn fail — NGOẠI LỆ duy nhất của luật "không hỏi user"): nhờ user SETUP MÔI TRƯỜNG 1 LẦN — đăng nhập Google/profile thường (+ mở panel hộ lần đầu) — KHÔNG phải nhờ mở hộ mỗi vòng; sau setup đúng, quay lại Cách 1 (các cách tự động thường sống lại — kỷ nguyên 1182 chạy tự động hoàn toàn trên profile thường)
   - Đường thoát CHIẾN LƯỢC nếu side panel mãi không click được: chuyển sang `gemini.google.com` mở trong TAB — web page thường nên automate được 100% bằng page()/CDP (xem pitfalls.md mục "Gemini web app"); Gem "Thẩm định đề thi" có trong web app, gửi URL đề trong prompt thay vì chia sẻ tab — cần user chạy thử 1 lần để xác nhận biên bản chấm qua URL đạt chất lượng như chấm tab share
   - Từ vòng lặp SAU (panel từng mở, Cmd+R làm mất sharing): KHÔNG toggle chip nữa — tìm `AXButton "Mở Gemini trong Chrome"`/nút re-share trong AX tree và click (nút này CÓ trong tree)
   - Verify: window title có "Bạn đang chia sẻ thẻ này với Gemini", HOẶC panel có text `Đang chia sẻ "<tên đề>"` / `AXWebArea "Gemini Chrome"` trong get_window_state

2. **Bắt đầu cuộc trò chuyện mới** (nếu có lịch sử cũ):
   - `get_window_state(max_elements=5000)` → tìm `AXButton "Bắt đầu cuộc trò chuyện mới"`
   - Click nó (AXPress).

3. **Gọi skill** — đọc placeholder ô input để chọn ĐÚNG MỘT luồng (KHÔNG trộn `/` với `@`):
   - `press_key("f6", FG)` focus input
   - Placeholder "Nhập nội dung / để sử dụng kỹ năng" → **luồng `/`**: `type_text("/", FG)` (verified:true = vào đúng ô) → `get_window_state` tìm `AXMenuItem "Thẩm định đề thi - dethimontoan.net"` → click
   - Placeholder "Nhập @ để hỏi về một thẻ" (bản Gemini mới) → **luồng `@`**: `type_text("@Thẩm định đề thi - dethimontoan.net", FG)` → get fresh state, menu mention hiện mục skill → click (menu không click được mới `press_key("return")` chọn mục đang highlight)
   - Verify skill đã thành CHIP vàng trong ô input RỒI MỚI Click `AXButton "Gửi"` — gõ nguyên tên skill + Enter khi chưa có chip = gửi tin nhắn thường, skill KHÔNG kích hoạt
   - **LUÔN get fresh state trước mỗi click** — element index thay đổi mỗi snapshot

4. **Chờ kết quả — Poll 15s** (không sleep 120):
   - `get_window_state` mỗi 15s, tìm "BIÊN BẢN THẨM ĐỊNH" + "Gemini là một AI" trong result
   - Tối đa 20 lần (5 phút). Nếu quá → reload tab + chạy lại.
   ```python
   from hermes_tools import mcp__cua_driver__get_window_state as get_window_state
   import time
   for _ in range(20):  # tối đa 5 phút
       time.sleep(15)
       r = get_window_state(pid=..., window_id=..., max_elements=2000, query="BIÊN BẢN|Gemini là một AI")
       if 'BIÊN BẢN THẨM ĐỊNH' in str(r) and 'Gemini là một AI' in str(r):
           break  # Gemini đã trả xong
   ```

### Bước 3 - Trích xuất biên bản (NGUYÊN VĂN)

**Phương pháp 1 (khuyến nghị) — từ structured elements**:
Dùng kết quả của `get_window_state(max_elements=2000, query="BIÊN BẢN|Mã đề")`. Biên bản nằm trong `elements[].label` của các node `AXStaticText` dưới Gemini panel. Ghép các label kề nhau từ "BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ:" đến "Gemini là một AI".

**Phương pháp 2 — từ file dump (fallback)**:
- **NGUYÊN VĂN, KHÔNG RÚT GỌN**
- KHÔNG dùng nút "Sao chép" (AXPress không trigger clipboard)
- Script parse dump + dấu hiệu nhận biết element: `references/extract-script.md`

**Định dạng biên bản chuẩn** (checklist xác nhận extract ĐỦ trước khi gửi):
- Dòng đầu: `BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ: XXXX)` (có thể kèm `- LẦN N`)
- Đúng 10 dòng `Tiêu chí k (…): ĐẠT/LỖI - …`
- Có LỖI → PHẢI có mục `ĐỀ XUẤT SỬA ĐỔI` (mc: field `q/o/c/e`; sa: `q/c/e` với barem `Bước k (0,25đ):`)
- ⚠️ Map field khi apply: biên bản ghi `c:` cho đáp án sa nhưng data JSON dùng `ans` (chỉ mc dùng `c`) — thực tế 1182: biên bản `c: 90` → code `'ans' => '90'`
- Thiếu phần nào → extract lại, KHÔNG gửi biên bản cụt

### Bước 4 - Claude Code tạo PHP patch

**PHƯƠNG PHÁP 1 — Claude Code Web Remote (ƯU TIÊN)**:
1. Trích xuất biên bản → `/tmp/bienban_XXXX.txt` (NGUYÊN VĂN)
2. Copy vào repo: `cp /tmp/bienban_XXXX.txt /tmp/dethimontoan.net/raw_bienban_XXXX.txt`
   — MỖI ĐỀ CHỈ 1 FILE `raw_bienban_XXXX.txt`, vòng sau GHI ĐÈ vòng trước (khớp luật "chỉ dùng biên bản mới nhất"); KHÔNG tạo `bienban_XXXX.txt`/`_lanN.txt` mới (1182 từng để lại 4 file rác); số lần ghi ở dòng đầu biên bản + commit message
3. Commit lên GitHub: `git add raw_bienban_XXXX.txt && git commit -m "raw bien ban XXXX lan N" && git pull --rebase origin main && git push origin main`
4. Mở `https://claude.ai/code` trong Chrome
5. Click session "dethimontoan.net" trong sidebar
6. `type_text(instruction, FG)` vào Prompt → `press_key(return)` — KHÔNG BAO GIỜ FG cho Return (FG Return làm mất keyboard focus → gõ xong mà KHÔNG gửi); type_text/click/F6 thì dùng FG bình thường
7. Sidebar hiện "Running dethimontoan.net"
8. **Tự động poll 15s** kiểm tra sidebar còn Running không:
   - `get_window_state` query "Running dethimontoan" — nếu không còn → done
9. Kiểm tra commit mới trên GitHub sau khi done

**Yêu cầu mẫu cho Claude Code Web**:
```
Đọc raw_bienban_XXXX.txt (biên bản Gemini) và TẤT CẢ patch hiện có của đề: inc/patch_XXXX*.php
(có thể có file revision riêng như patch_XXXX_r2.php). Cập nhật patch để khớp chính xác đề xuất
Gemini. Giữ NGUYÊN các fix cũ (r1, r2...). Bump option nếu cần. Commit push.
```

**PHƯƠNG PHÁP 2 — Claude Code CLI (FALLBACK)**:
```bash
export PATH="$HOME/.local/bin:$PATH"
cd /tmp/dethimontoan.net
claude auth status --text || claude auth login
claude -p '...' --allowedTools "Read,Write,Edit,Bash" --max-turns 40
```
⚠️ CLI timeout 300s (mặc định). Cần `pty=true` trong terminal() vì Claude CLI cần PTY. Dùng `--allowedTools "Read,Write,Edit,Bash"` (tool chạy lệnh tên là "Bash", không phải "Execute"; không thêm "WebFetch" — WebFetch hay bị lỗi permission). Nếu session chặn git write, tự tay `git add/commit/push` sau. `claude -p` có thể TREO >180s khi đọc raw_bienban (đề 1183 đã gặp) → đừng đợi quá 5 phút, chuyển Web Remote hoặc tự tạo patch fallback.

**⚠️ Quản lý tab khi dùng Web Remote**:
- Tab claude.ai/code KHÔNG navigate đi nơi khác
- Khi cần navigate exam URL: dùng TAB KHÁC hoặc Cmd+L
- page() navigate TAB HIỆN TẠI → chỉ dùng khi tab đang ở exam

**Quy ước patch (đúc kết từ patch_1182/1183 chạy thật)**:
- Hàm + gate option TRÙNG TÊN dạng `ttp_patch_<mã>_rN_v1` (hook `wp_loaded`, có option → bỏ qua)
- Revision mới: thêm hàm rN vào `inc/patch_XXXX.php` (1182 gom r1–r3 một file) hoặc file riêng `inc/patch_XXXX_rN.php` (1183 r2 — đường fallback)
- Sửa lại revision ĐÃ áp trên server: GIỮ tên hàm, BUMP hậu tố option (`_r2c_v1` → `_r2d_v1`) để chạy lại; guard theo NỘI DUNG MỚI ⇒ idempotent
- Guard phải UNIQUE với nội dung mới, KHÔNG trùng nội dung cũ (guard `AH² = BH·CH` từng khớp nhầm lời giải cũ → barem không áp); sa đổi cả câu → guard theo `q`, chỉ thêm barem → guard theo `e` — bảng chi tiết: `references/patch-loop-sa-bug.md`
- ⚠️ TRƯỚC khi thêm hàm/file mới: `grep -rn "ttp_patch_<mã>" inc/` — trùng tên hàm → Fatal "Cannot redeclare" → 500 TOÀN SITE (`php -l` KHÔNG bắt được trùng tên liên-file)

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
from hermes_tools import terminal, mcp__cua_driver__get_window_state as get_window_state
import time

for _ in range(40):  # Tối đa 10 phút (40 lần x 15s)
    time.sleep(15)
    r = get_window_state(pid=..., window_id=..., max_elements=100, query="Running dethimontoan")
    if 'Running' not in r['result']: break  # Done

# Kiểm tra commit mới — có thể có SESSION THỨ HAI cùng push (GitHub auto-deploy) → LUÔN pull trước khi verify
terminal("cd /tmp/dethimontoan.net && git pull && git log --oneline -3")
```

### Bước 5 - Reload → Gemini lại
- Nếu đang ở claude.ai/code: chuyển tab exam (dùng tab khác), reload Cmd+R
- Nếu đã ở exam tab: Cmd+R reload
- Sau Cmd+R Gemini thường MẤT tab sharing → mở lại panel (Bước 2.1) rồi mới gọi skill
- Đợi ~5s → Bước 2 (bỏ qua "Bắt đầu cuộc trò chuyện mới" nếu Gemini vẫn đang share)

### Bước 6 - Chuyển đề tiếp theo
- Chuỗi ĐẠT chính xác (tín hiệu DUY NHẤT): **"TOÀN BỘ ĐỀ [MÃ] ĐÃ ĐẠT CHUẨN ĐỘC BẢN, CHÍNH XÁC VÀ ĐẢM BẢO TÍNH TRỰC QUAN. SẴN SÀNG PHÁT HÀNH."** → ✅ Đề này OK. Cập nhật trạng thái trong `references/exam-list.md`: ✅ CHỈ khi chuỗi ĐẠT được thấy trên SITE ĐÃ DEPLOY; fix xong nhưng còn chờ deploy → đánh 🔄, phải quay lại verify trước khi sang đề mới.
  - **KHÔNG dừng.** Chuyển sang đề kế tiếp trong danh sách.
  - Quay lại **Bước 1** (navigate URL đề mới).
- **Còn bất kỳ LỖI nào** → quay lại Bước 3 (extract biên bản) → Bước 4 (Claude Code fix).
  - **Cứ lặp fix→check cho đến khi Gemini báo ĐẠT mới chuyển đề.**
  - Không bỏ qua lỗi nào.
- **CẢNH BÁO TC9 (barem)**: Nếu Gemini báo LỖI TC9 nhưng barem đã thêm → kiểm tra deploy. Chưa deploy = Gemini thấy đề cũ.

## Pitfalls

### Gemini panel không mở
- Thử TRƯỚC: `hotkey(["cmd","shift","y"], FG)` — toggle panel không cần toạ độ/AX (references/gemini-panel-methods.md, Phương pháp 2 đã kiểm chứng).
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
- Nguyên nhân: type_text vào SAI element (Claude có 2+ AXWindow → nhiều Prompt textarea), hoặc DÙNG FG cho press_key(return) — FG Return làm mất keyboard focus → gõ xong mà không gửi (kiểm chứng 2026-07-21)
- **Fix**: reload tab claude.ai/code, click ĐÚNG session "dethimontoan.net" (không prefix Error/Running), type_text FG lại, press_key("return") KHÔNG FG
- KHÔNG xoá conversation cũ — chỉ gõ instruction mới; session "Error" → tạo New session rồi gửi lại

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
- **NGUYÊN TẮC**: Gửi cho Claude Code PHẢI là **nội dung ĐẦY ĐỦ và CHÍNH XÁC 100%** của biên bản thẩm định Gemini. **TUYỆT ĐỐI KHÔNG rút gọn, tóm tắt, hay diễn giải lại biên bản.**
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

### Biên bản bị RỚT ký tự khi extract (AX label)
- Dấu nhân `·` và đoạn đứng sau dấu ngoặc kép có thể bị rớt/đứt dòng khi ghép label — thực tế 1182 lần 5: `AB^2 = BH` ⏎ `BC` (mất `·`), dòng TC5/TC10 đứt giữa câu
- Vẫn gửi NGUYÊN VĂN phần trích được (không tự "vá" lại biên bản)
- Claude Code khi APPLY phải khôi phục ký tự rớt theo ngữ cảnh toán (patch 1182 đã viết đúng `AH² = BH·CH`), KHÔNG copy nguyên chỗ đứt vào nội dung đề

### Chrome window bị stale
- `list_windows` → lấy window_id mới nếu cũ không hoạt động
- `bring_to_front` để focus window

### Toggle panel "fail" liên tục khi có NHIỀU cửa sổ Chrome (đã vấp 21/7)
- Triệu chứng: thử đủ hotkey/AXPress/popup/pixel — verify vẫn báo panel không mở
- Nguyên nhân: 10 cửa sổ Chrome — action đánh vào cửa sổ ACTIVE, verify đọc window_id KHÁC → false negative; và vì "fail mới sang cách kế", chuỗi cách thành toggle MỞ-ĐÓNG-MỞ-ĐÓNG
- Fix: gom về 1 cửa sổ; trước mỗi action bring_to_front + xác nhận frontmost; action và verify cùng window_id; screenshot xác nhận sau mỗi toggle
- Kẹt thật sau 2 vòng đủ cách → Cách 5: nhờ user mở thủ công 1 lần (ngoại lệ luật "không hỏi user")

## Tham khảo (references/)
Đọc ĐÚNG file khi cần, đừng load tất cả:
- `exam-list.md` — danh sách đề đầy đủ + trạng thái ✅/⏳ (CẬP NHẬT khi xong mỗi đề)
- `gemini-panel-methods.md` — 3 cách mở panel Hỏi Gemini (khi Bước 2 kẹt)
- `chrome-ax-menu-bar.md` — AX tree bị menu bar chiếm (get_window_state toàn AXMenuBar)
- `extract-script.md` — script parse biên bản từ dump + dấu hiệu element (Bước 3)
- `paste-methods.md` — thứ tự phương pháp gửi biên bản + snippet trích xuất (Bước 4)
- `patch-loop-sa-bug.md` — bug sa chỉ apply `e` + luật chọn guard (khi viết/duyệt patch)
- `pitfalls.md` — pitfall UI tổng hợp (URL bar, session sai, Chrome pid, guest mode…)
- `session-learnings-*.md` — nhật ký từng phiên; MỚI NHẤT: `20260721-v4` (đề 1183) + `20260721-v3` (đề 1182); file cũ hơn chỉ để tra lịch sử, mâu thuẫn thì file mới thắng

## Luật bất biến
- **GỬI NGUYÊN VĂN, KHÔNG RÚT GỌN DÙ 1 TỪ**
- **KHÔNG hỏi user** — tự retry/reset, KHÔNG hỏi "có muốn tiếp tục không" (ngoại lệ DUY NHẤT: blocker UI cứng theo Cách 5 Bước 2.1 — nhờ thao tác thủ công 1 lần rồi tự chạy tiếp)
- **Reload tab trước mỗi lần chạy Gemini**
- **Luôn get fresh state trước click**
- **Tự động poll 15s — không sleep 120s**
- **Sau Claude Code: verify output khớp biên bản gốc**
- **press_key(return) vào Claude Code: KHÔNG BAO GIỜ FG**
- **ĐẠT → chuyển đề kế tiếp, không dừng**
