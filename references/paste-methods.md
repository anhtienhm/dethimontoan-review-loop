# Phương pháp gửi biên bản (ưu tiên từ cao đến thấp) — KIỂM CHỨNG 2026-07-21

## Ưu tiên 1: Claude Code CLI (LUỒNG CHÍNH MỚI)

```bash
export PATH="$HOME/.local/bin:$PATH"
cd /tmp/dethimontoan.net
cp /tmp/bienban_XXXX.txt ./
claude -p 'Đọc bienban_XXXX.txt (biên bản). Tạo inc/patch_XXXX.php theo mẫu inc/patch_1181.php.'
```

Claude Code CLI:
- Đọc được local file (không giống Claude.app)
- Tạo PHP patch trực tiếp (code, verify, write)
- Không bị timeout
- Không bị WebKit chặn paste

## Ưu tiên 2: Claude Code Web Remote

Mở `https://claude.ai/code` → click session "dethimontoan.net" → type → Send.

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
d = json.loads(open('/path/to/result.txt'))
result = d['result']
start = result.find('BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ:')
end = result.find('Đang chia sẻ', start)
out = [l.split('AXStaticText = ',1)[1].rsplit('" [actions',1)[0].lstrip('"') 
       for l in result[start:end].split('\\n') if 'AXStaticText = ' in l]
open(f'/tmp/bienban_{ma}.txt', 'w').write('\\n'.join(out))
```

## User preferences (CRITICAL)
- **GUỬI NGUYÊN VĂN, KHÔNG RÚT GỌN** — user phàn nàn 5+ lần
- **GUỬI RAW, KHÔNG THÊM PREFIX**
- **Tự động, không hỏi user**
