# Key technical learnings from 2026-07-20 session

## Copy/Paste mechanism (CRITICAL)

### Sao chép button DOES NOT WORK
- AXPress on Gemini's "Sao chép" button does NOT trigger `navigator.clipboard.writeText()` 
- Clipboard stays empty after AXPress click
- Pixel click at correct coordinates also fails
- **SOLUTION**: Extract biên bản text directly from AX tree dump

### Cmd+V DOES NOT WORK into Claude.app webview
- Claude.app uses Electron/WebKit webview
- Cmd+V (both via cua-driver hotkey and osascript) is blocked
- **SOLUTION**: type_text with "|" separators, 700-char chunks (fallback)
- **NEW PREFERRED SOLUTION**: Save to file, tell Claude to read it

### set_value DOES NOT WORK on web content
- set_value on AXTextArea writes AXProperty but WebKit ignores it for contenteditable areas
- Returns "success" but text doesn't appear in the visual input

## Chrome AX tree issues

### Apple menu stuck
- Symptom: get_window_state returns only AXMenuBar elements, no window content
- Fix: `killall SystemUIServer; sleep 2; osascript -e 'tell app "Google Chrome" to activate'`
- Alternative: `killall Dock` (but this is more disruptive)

### Chrome window disappears
- Symptom: list_windows shows fewer Chrome windows, page tool errors
- Cause: Claude Code may navigate Chrome to cPanel or close tabs
- Fix: Use launch_app to open new Chrome or find surviving window

## Gemini interaction

### Skill activation
- MUST use "/" -> menu -> CLICK skill item (NOT just type full text + Enter)
- AXMenuItem "Thẩm định đề thi - dethimontoan.net" appears in drop-down 
- After clicking skill, chip shows in textarea, then click "Gửi"

### Element indices vary
- Textarea indices change after every reload/navigation
- Always do fresh get_window_state before clicking

## Claude.app interaction

### Session selection
- Must click "dethimontoan.net" in sidebar before typing
- Watch for "Error" prefix vs "Running" prefix vs no prefix
- "Error" sessions need new session or existing clean one

### File-based instruction (NEW - PREFERRED)
- Single instruction: "Fix đề Mã XXXX. Đọc /tmp/bienban_XXXX.txt để xem biên bản chi tiết."
- ~100 chars, type_text confirms in 3s
- Claude reads file directly, has full exact biên bản
- No risk of truncation, summarization, or timeout
