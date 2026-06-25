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