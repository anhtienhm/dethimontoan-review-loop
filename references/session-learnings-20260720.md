# Session Learnings — 2026-07-20

## Critical: NEVER Summarize Gemini Biên Bản
User corrected this 5+ times this session. The biên bản sent to Claude MUST be verbatim from Gemini, only replacing newlines with " | ". Any summarization (even slightly condensing TC descriptions) is rejected.

## Sao Chép Button Does NOT Work via AXPress
The "Sao chép" button in Gemini uses JavaScript `navigator.clipboard.writeText()` which requires a real physical user gesture. AXPress on the button does NOT trigger the clipboard API. Clipboard remains empty after AXPress.

**Verified workaround**: Extract biên bản text from AX tree dump, save to file, then `pbcopy`.

## Cmd+V Does NOT Work in Claude.app Webview
Claude.app is an Electron app that blocks synthetic paste events. Neither `hotkey(cmd+v)` nor `osascript keystroke "v" using command down` works.

**Verified workaround**: `type_text` in chunks of 600-700 characters with 30ms delay, using " | " as newline separator. type_text >900 chars risks 300s timeout.

## Apple Menu Blocks Chrome AX Tree
When the macOS Apple menu is open (even without visual indication), Chrome's entire AX tree is consumed by the menu bar, making the Gemini panel and web content invisible to accessibility tools.

**Fix**: `killall SystemUIServer; sleep 2; osascript -e 'tell app "Google Chrome" to activate'`
Or: `osascript -e 'tell app "Finder" to activate' -e 'delay 1' -e 'tell app "Google Chrome" to activate'`

## Session Selection in Claude
Claude may have multiple sessions. The target is "dethimontoan.net" WITHOUT any prefix (not "Running dethimontoan.net", not "Error dethimontoan.net"). If all sessions have prefixes, create a "New session".

Session also has 2 AXWindows (sidebar + workspace view). "Open session dethimontoan.net" button is in the workspace view's session list (element ~410).

## Chrome Profile / Guest Mode
Chrome may open in guest profile ("Vô danh") where extensions are available but behave differently. The window title ends with "- Google Chrome - Vô danh". Use regular (non-guest) Chrome when possible.

## Tab Management After Claude Fix
Claude navigates to cPanel during fix operations, which replaces/removes the dethimontoan.net tab. Use `page(execute_javascript)` CDP to navigate any existing Chrome tab back to the exam URL:
```javascript
window.location.href = 'https://dethimontoan.net/de-thi-giua-hoc-ky-1-mon-toan-lop-8-XXXX/'
```

## Gemini Panel State After Reload
After `Cmd+R` reload, Gemini loses tab sharing. Must re-share:
- Click AXButton "Mở Gemini trong Chrome"
- If same Gemini session: click "Bắt đầu cuộc trò chuyện mới" before typing "/"

## type_text Timeout
type_text with ~800 chars timed out at 300s. The CGEvent path has overhead beyond the 30ms delay. Keep chunks to 600-700 chars maximum.

## Addressing Bar
When navigating via address bar, ALWAYS Cmd+A → Delete first. Otherwise type_text appends to existing URL, creating concatenated invalid URLs.

## Gemini Menu Not Appearing
When typing "/" doesn't trigger the skill menu, the issue is often:
1. Wrong textarea clicked (there may be 2 textareas)
2. Gemini panel not sharing the tab
3. Need to click "Bắt đầu cuộc trò chuyện mới" first
