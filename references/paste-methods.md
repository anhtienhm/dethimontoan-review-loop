# Phương pháp gửi biên bản (ưu tiên từ cao đến thấp) — KIỂM CHỨNG 2026-07-21 (cập nhật cuối ngày: Web Remote lên Ưu tiên 1 — khớp SKILL.md)

## Ưu tiên 1: Claude Code Web Remote (LUỒNG CHÍNH)

Mở `https://claude.ai/code` → click session "dethimontoan.net" → type → Send.
Ổn định hơn CLI — CLI từng treo >180s khi đọc raw_bienban_1183 (xem `session-learnings-20260721-v4.md`).

## Ưu tiên 2: Claude Code CLI

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /tmp/dethimontoan.net
cp /tmp/bienban_XXXX.txt ./
claude -p 'Đọc bienban_XXXX.txt (biên bản). Tạo inc/patch_XXXX.php theo mẫu inc/patch_1181.php.'
```

Claude Code CLI:
- Đọc được local file (không giống Claude.app)
- Tạo PHP patch trực tiếp (code, verify, write)
- Không bị WebKit chặn paste
- ⚠️ Có thể TREO >180s khi đọc raw file → đặt timeout, treo thì quay về Web Remote hoặc tự tạo patch

## Ưu tiên 3: Claude.app (FALLBACK - chỉ khi Claude Code không available)

### set_value
- `set_value(element_index, value=full_text)` trên AXTextArea Prompt
- `press_key(return)` để gửi
- Có thể bị WebKit ignore

### type_text (FALLBACK của Claude.app)
- Chunks ~950 ký tự, " | " separator
- ~30s/chunk

### KHÔNG DÙNG
- **Cmd+V** — Claude.app webview chặn paste
- **File-based** ("Đọc /tmp/bienban_XXXX.txt") — Claude.app không đọc được local file
- **Nút "Sao chép"** — AXPress không trigger clipboard API

## Trích xuất biên bản
```python
import json
d = json.load(open('/path/to/result.txt', encoding='utf-8'))
result = d['result']
start = result.find('BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ:')
end = result.find('Đang chia sẻ', start)
out = [l.split('AXStaticText = ', 1)[1].rsplit('" [actions', 1)[0].lstrip('"')
       for l in result[start:end].split('\n') if 'AXStaticText = ' in l]
open('/tmp/bienban_XXXX.txt', 'w').write('\n'.join(out))
```

## User preferences (CRITICAL)
- **GỬI NGUYÊN VĂN, KHÔNG RÚT GỌN** — user phàn nàn 5+ lần
- **GỬI RAW, KHÔNG THÊM PREFIX**
- **Tự động, không hỏi user**
