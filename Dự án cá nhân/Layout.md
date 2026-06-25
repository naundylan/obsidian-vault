Cấu trúc chi tiết của SeatLayout: 
Đối với bảng **`seat_layouts`** (lưu sơ đồ mẫu):

- `id`: `UUID` (Primary Key).
- `name`: `VARCHAR(150)` (Tên sơ đồ).
- `venueName`: `VARCHAR(150)` (Tên địa điểm diễn ra sự kiện).
- `status`: `VARCHAR(20)` (Lưu Enum `DRAFT`, `PUBLISHED`, `ARCHIVED`).
- `workspace_rows` / `workspace_cols`: `INTEGER` (Kích thước lưới thiết kế).
- `used_rows` / `used_cols`: `INTEGER` (Diện tích lưới thực tế chứa ghế, phục vụ cho việc tính toán căn giữa).
- `seat_count`: `INTEGER` (Tổng số ghế đã vẽ).
- `layout_data`: **`JSONB`** (Trọng tâm lưu trữ cấu trúc bản vẽ).




Cấu trúc hoàn chỉnh của 1 layoutData
{
	"workspaceRows": 50,
	"workspaceCols": 50,
	"templateType": "HALL_RECTANGLE",
	"usedBounds": {
	"minRow": 8, "maxRow": 20,
	"minCol": 6, "maxCol": 44,
	"usedRows": 13, "usedCols": 39
},
"cells": [
	{
		"row": 8,
		"col": 6,
		"previewLabel": "A1",
		"ticketClassKey": "vip",
		"customPreviewLabel": false
	}
],
"decorations": [
	{
		"id": "hall_rectangle-stage",
		"type": "stage",
		"label": "Stage / Screen",
		"row": 2,
		"col": 15,
		"rowSpan": 3,
		"colSpan": 20,
		"shape": "rect"
	}
]
}



Trước khi có grid, màn canva, các chế độ, các số liệu chúng ta cần config các biến trước:
```
const CELL_SIZE = 30 // 1 ô bằng 30px
const WORKSPACE_BLOCK_SIZE = 50 // kích cỡ của 1 workspace
const DEFAULT_WORKSPACE = WORKSPACE_BLOCK_SIZE
const MAX_HISTORY = 50
const STANDARD_CLASS = { key: 'standard', name: 'Standard', color: '#2563eb' }
const SEAT_COUNT_PRESETS = [
  100, 200, 300, 400, 500, 600, 700, 800, 900, 1000,
  1500, 2000, 2500, 3000, 3500, 4000, 4500, 5000,
]
```



## 3. Cơ chế hoạt động của các Chế độ Vẽ (Paint), Xóa (Erase) và Kéo (Pan)

Việc chuyển đổi giữa các Mode vẽ dựa trên các hàm xử lý sự kiện chuột của Konva `<Stage>` và các hàm lắng nghe sự kiện của đối tượng `window` trong React.

```
[Mouse Down] (Lấy điểm bắt đầu) -> [Mouse Move] (Tính toán khoảng vẽ tạm thời) -> [Mouse Up] (Lưu/Xóa chính thức vào State)
```


### a. Chuyển đổi tọa độ Chuột sang Ô lưới (Pixel -> Grid Cell)

Mỗi khi chuột click hoặc di chuyển trên Stage, ta cần biết chuột đang nằm ở dòng nào, cột nào của lưới. Thuật toán chuyển đổi trong clientPointToCell như sau:

row=⌊clientY−rect.top−stageState.ystageState.scale×CELL_SIZE⌋row=⌊stageState.scale×CELL_SIZEclientY−rect.top−stageState.y​⌋

col=⌊clientX−rect.left−stageState.xstageState.scale×CELL_SIZE⌋col=⌊stageState.scale×CELL_SIZEclientX−rect.left−stageState.x​⌋

- `rect.left`, `rect.top`: Vị trí của khung canvas trên trình duyệt.
- `stageState.x`, `stageState.y`: Tọa độ dịch chuyển (offset) khi người dùng kéo màn hình (Pan).
- `stageState.scale`: Tỷ lệ thu phóng (Zoom).
- `CELL_SIZE`: Kích thước pixel của 1 ô (trong code là `30px`).


### a. Chế độ vẽ đường thẳng (Line Mode)

- Tính toán độ lệch dòng và cột giữa điểm bắt đầu và điểm kết thúc: `rowDelta = abs(end.row - start.row)` và `colDelta = abs(end.col - start.col)`.
- Nếu `colDelta >= rowDelta`: Vẽ đường ngang. Giữ cố định dòng `start.row`, chạy cột từ `minCol` đến `maxCol`.
- Nếu `rowDelta > colDelta`: Vẽ đường dọc. Giữ cố định cột `start.col`, chạy dòng từ `minRow` đến `maxRow`.



### b. Chế độ vẽ hình chữ nhật (Rectangle Mode)

- Xác định hình chữ nhật bao phủ bằng cách lấy cực trị của dòng và cột:
    - `minRow = min(start.row, end.row)`
    - `maxRow = max(start.row, end.row)`
    - `minCol = min(start.col, end.col)`
    - `maxCol = max(start.col, end.col)`
- Chạy 2 vòng lặp lồng nhau để quét qua toàn bộ tọa độ:
    
    javascript
    
    for (let row = minRow; row <= maxRow; row++) {
    
      for (let col = minCol; col <= maxCol; col++) {
    
        cells.push({ row, col });
    
      }
    
    }
    
- Tất cả các ô nằm trong tập hợp này sẽ được thêm mới (hoặc xóa đi) khi người dùng thả chuột.


	