# Trich xuat bien ban tu AX tree Gemini

## Van de
Nut "Sao chep cau tra loi" trong Gemini dung JavaScript `navigator.clipboard.writeText()`.  
Khi click bang AXPress, JavaScript event handler KHONG duoc trigger -> clipboard rong.  
**KHONG DUNG nut Sao chep.**

## Giai phap
Trich xuat truc tiep tu AX tree dump cua Chrome/Gemini panel bang script Python.

## Script

```python
import json

# 1. Doc file result tu get_window_state
data = json.load(open('/path/to/hermes-results/call_00_XXXX.txt', encoding='utf-8'))
lines = data['result'].split('\n')

out = []
capture = False

for l in lines:
    # Bat dau bien ban — marker PHAI CO DAU (text tren UI Gemini co dau tieng Viet)
    if 'BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ: XXXX)' in l and 'AXStaticText' in l:
        if not capture:
            capture = True
        else:
            break  # Dong thu 2 -> ket thuc
    
    if capture:
        if 'AXStaticText = ' in l:
            txt = l.split('AXStaticText = ', 1)[1].rsplit('" [actions', 1)[0].lstrip('"')
            
            # Stop khi gap dong khong lien quan (marker co dau — khop text that tren UI)
            # 'Gemini là một AI' = disclaimer cuoi ket qua — DIEM DUNG CHUAN (khop SKILL.md Buoc 3),
            # khong cho lot vao bien ban; con thieu marker nay thi khi dump khong co ban trung
            # script se quet het ca footer
            if 'Gemini là một AI' in txt or 'Bạn là Tổ trưởng' in txt or 'Thẩm định đề thi' in txt:
                break
            
            out.append(txt)

# Luu ra file
full_text = '\n'.join(out)
open('/tmp/bienban_XXXX.txt', 'w').write(full_text)
print(f'{len(out)} lines saved')
```

## Cach dung
1. Goi `get_window_state(max_elements=5000)` tren Chrome window co Gemini panel
2. Dung `python3 << 'PYEOF' ... PYEOF` de chay script tren result file
3. Output: `/tmp/bienban_XXXX.txt`
4. Kiem tra: `head -5 /tmp/bienban_XXXX.txt`

## Phương pháp 2 (khuyến nghị): Query trực tiếp từ get_window_state

Thay vì parse file dump, dùng `query` parameter của `get_window_state`:

```python
# 1. Gọi get_window_state với query để locate biên bản
# get_window_state(pid=..., window_id=..., max_elements=2000, query="BIÊN BẢN THẨM ĐỊNH")
# Output chứa structuredContent.elements với các element có label chứa nội dung biên bản.

# 2. Trong response, tìm tất cả element có label chứa "BIÊN BẢN" hoặc "Tiêu chí" hoặc "ĐỀ XUẤT"
# rồi ghép các label đó lại.

# 3. Viết file
with open('/tmp/bienban_XXXX.txt', 'w') as f:
    f.write(full_text)
```

## Cách dùng (modern)
1. Gọi `get_window_state(max_elements=2000, query="BIÊN BẢN|Mã đề", pid=chrome_pid, window_id=chrome_window_id)`
2. Duyệt `result.structuredContent.elements[]` — element có `label` chứa "BIÊN BẢN THẨM ĐỊNH" là điểm bắt đầu
3. Các element `AXStaticText` kề nhau dưới cùng `parent_index` chứa nội dung biên bản
4. Ghép `label` của các element đó lại (ngăn cách bằng newline hoặc space)
5. Lưu ra `/tmp/bienban_XXXX.txt`

## Dấu hiệu nhận biết biên bản trong elements
- `label` bắt đầu bằng "BIÊN BẢN THẨM ĐỊNH (MÃ ĐỀ:" → header
- `label` bắt đầu bằng "Tiêu chí N (" → từng tiêu chí
- `label` bắt đầu bằng "ĐỀ XUẤT SỬA ĐỔI" → đề xuất
- `label` bắt đầu bằng "Câu N Phần" → từng đề xuất cụ thể
- `label` chứa "Gemini là một AI" → điểm kết thúc
