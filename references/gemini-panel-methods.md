# Mở Hỏi Gemini — 3 phương pháp đã kiểm chứng

## Vị trí nút Hỏi Gemini (đo từ screenshot 21/7, cửa sổ 2000px)
- Chip ✦ "Hỏi Gemini" nằm ở **hàng tab strip** (cùng hàng các tab và nút "+", dưới menu bar macOS, TRÊN toolbar), **sát mép phải cửa sổ Chrome**.
- Tâm chip ≈ **(W−85, 71)** theo toạ độ cửa sổ (W = bề rộng cửa sổ; 2000px → ~(1915, 71)); vùng chip ≈ x[W−157, W−10], y[55, 90].
- Hàng toolbar NGAY DƯỚI (y≈130): Tiện ích (W−152) · avatar (W−80) · ⋮ (W−32) — đừng click nhầm hàng.
- Icon ✦ THỨ HAI trên menu bar macOS (~x1357, y22) là `AXMenuBarItem [help="Bật/tắt Gemini trong Chrome"]` — chỉ dùng AXPress, không pixel click.
- Chip KHÔNG xuất hiện trong AX tree (get_window_state) khi panel chưa mở — không thấy element ≠ không có extension.
- Click pixel thường chạm "background element" — CHỤP SCREENSHOT xác định tâm rồi click, không click mù; ưu tiên Cmd+Shift+Y (Phương pháp 2) trước.

## Phương pháp 1: Click trực tiếp
Thử click ở các tọa độ window-local: x=1440-1460, y=35-50.
Nếu hit-test trả "background element", thử scope="desktop" với tọa độ screen-absolute.

## Phương pháp 2: Cmd+Shift+Y
`hotkey(keys=["cmd","shift","y"], delivery_mode="foreground")` — toggle panel.
Kiểm tra window title có "Bạn đang chia sẻ thẻ này với Gemini".

## Phương pháp 3: CDP/Playwright (nếu có CDP port)
`page.click('text="Hỏi Gemini"')` — tìm button theo text.

## Sau khi panel mở
- type_text có verified:true = đã focus đúng ô input Gemini.
- type_text unverified = chưa focus được, cần F6 thêm.
- Panel content CÓ trong AX tree: dùng `get_window_state(max_elements=2000, query="BIÊN BẢN|Mã đề|cần kiểm tra")` trên pid Chrome.
  - Kết quả Gemini nằm trong `structuredContent.elements[].label` từ các node `AXStaticText` dưới `AXWebArea "Gemini Chrome :: Cuộc trò chuyện mới"`.
  - Dùng `query` parameter để filter cây → chỉ lấy nodes có chứa text cần tìm.
  - Cần `max_elements=2000` vì cây Gemini thường >1500 nodes.
- Nội dung biên bản được phát tán qua nhiều element label rời rạc (mỗi cụm text là 1 element) → cần ghép chuỗi từ các element kề nhau.
- Nếu chưa thấy kết quả dù đã set max_elements=2000: poll mỗi 15s (tối đa 20 lần = 5 phút).
