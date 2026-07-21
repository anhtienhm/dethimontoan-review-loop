# Session Learnings 2026-07-21-v3

## Tổng quan Mã 1182 — Hoàn thành
- **7 commit fix**, **5 lần thẩm định Gemini**, cuối cùng ĐẠT
- r1: TC3/TC6/TC7 (eacdc1a)
- r2b: TC4 Câu 7 + sa4 (255bf4f)
- r2c: barem sa1/sa2 + fix guard (1517ce5)
- r2d: sa1→tivi 72×54 + barem pr6 (c3f8bd3 — từ Claude Code auto)
- r3: mc10 tỉ số lượng giác (b09a0d8)
- Auto-merge: f53f323

## Luật bất biến từ user
- **KHÔNG hỏi "có muốn tiếp tục không"** — user nói "đừng hỏi lại tôi có muốn tiếp tục hay không"
- **KHÔNG rút gọn biên bản** — user nói "tôi muốn bạn copy đầy đủ biên bản luôn chứ không được rút ngắn"
- **Tự động poll** — "tự động check claude code fix và gemini chấm sau mỗi 15 giây"
- **Tự động reload khi lag** — "nếu bị lag thì load lại tab rồi gõ yêu cầu tiếp tục"

## Gemini extension via Tiện ích popup
- User has "Gemini in Chrome" extension (not built-in sidebar)
- Must open via "Tiện ích" (Extensions) button, not pixel click
- Located at AXPopUpButton "Tiện ích" in Chrome toolbar
- Then find AXButton "Hỏi Gemini" in the popup (~index 1598)
- After clicking, the sidebar opens and page sharing begins
- Close popup with Escape before proceeding
- Verify: window title phải có "Bạn đang chia sẻ thẻ này với Gemini"

## Claude Code Web Remote — press_key Return pitfall
- NEVER use delivery_mode="foreground" with press_key("return") in Claude Code
- FG Return makes keyboard focus disappear → text typed but not sent
- Sidebar shows "Idle" but message never arrived
- Fix: press_key("return") with default (background) mode
- Other actions (type_text, click, F6) can use FG

## Claude Code lag recovery
- If sidebar shows "Running" for >60s without visible progress, or stays "Idle" after sending:
  1. Reload tab (Cmd+R, delivery_mode="foreground")
  2. Wait for page to load (~5s)
  3. Click session in sidebar
  4. type_text(instruction, FG)
  5. press_key("return") — NO FG
- Do NOT delete old conversation — just type new instruction

## Poll 15s pattern for Gemini + Claude Code
- Gemini: check window result file for "BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ:" + "Gemini là một AI"
- Claude Code: check sidebar for "Idle" instead of "Running"
- Both poll at 15s intervals, max 20-40 iterations (5-10 min)
- Use execute_code or terminal loop with time.sleep(15)

## Multiple Claude Code sessions possible
- GitHub auto-deploy can trigger a SECOND Claude Code session
- One session pushes from CLI, another pushes from Web Remote
- Creates merge commits via github-actions
- Always check git log HEAD..origin/main after Claude Code finishes
- Pull the latest before verifying

## Bug: patch only applies e, forgets q/ans for sa type
- Claude Code loop for sa type sometimes only writes 'e', ignores 'q' and 'ans'
- Guard logic: if sa has 'q' in target → guard by q; otherwise guard by e
- Fix: bump gate option (_r2_v1 → _r2b_v1) and fix loop to apply all three fields
- Verify by comparing raw_bienban with patch file field-by-field
- **Triệu chứng**: Gemini vẫn báo LỖI "7x=0" dù patch đã push → vì chỉ apply barem, không apply q mới

## Bug: guard trùng với lời giải cũ
- Guard 'AH² = BH·CH' CÓ trong lời giải 1 dòng cũ → guard khớp nhầm → barem KHÔNG áp
- Fix: đổi guard thành cụm CHỈ có trong barem mới ('hệ thức liên hệ giữa đường cao')
- Lesson: guard cần UNIQUE, không trùng với bất kỳ nội dung nào có sẵn

## Gemini skill menu: @ vs / prefix
- Newer Gemini extension versions use "@" (mention) instead of "/" (slash)
- Check input placeholder to determine:
  - "Nhập nội dung / để sử dụng kỹ năng" → use "/"
  - "Nhập @ để hỏi về một thẻ" → use "@"
- When using @: type_text("@Thẩm định đề thi - dethimontoan.net", FG) → press_key("return")

## web_extract for deploy verification
- Use web_extract(URL) to check if patches deployed on server
- Fast, no Chrome needed
- Check specific content: Câu 4 text, Câu 1 text, đáp án values
- If content unchanged → webhook/git pull hasn't run yet

## Window staleness after Chrome crash/restart
- window_id can become stale → "No macOS window found for window_id N"
- Fix: list_windows(pid=chrome_pid, on_screen_only=True) → get new window_id
- Then use new window_id for all get_window_state/click calls
- Dùng mcp__cua_driver__get_accessibility_tree để xem danh sách window

## get_window_state trả về menu bar thay vì window
- Đôi khi get_window_state trả về app-level tree (AXMenuBar/AXApplication) thay vì window content
- Triệu chứng: elements = 71-200, toàn menu items (Apple, Chrome, Tệp, Sửa...)
- Không phải lỗi driver — thường do window đang trong trạng thái focus menu
- Fix: bring_to_front() → press_key("escape") → get_window_state lại
- Hoặc dùng desktop scope click để tương tác

## Exam pass criteria string
- Gemini outputs exact string: "TOÀN BỘ ĐỀ [MÃ] ĐÃ ĐẠT CHUẨN ĐỘC BẢN, CHÍNH XÁC VÀ ĐẢM BẢO TÍNH TRỰC QUAN. SẴN SÀNG PHÁT HÀNH."
- This is the ONLY signal that exam passed
- Any LỖI → loop continues
- Gemini có thể báo lỗi khác mỗi lần (TC4 lỗi cũ hết, lỗi mới xuất hiện) → luôn dùng biên bản mới nhất
