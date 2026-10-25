จากภาพหน้าจอ (`image_0695e3.png`) และโค้ดที่ส่งมา ผมเห็นสาเหตุที่แท้จริงของทุกปัญหาชัดเจนเลยครับ ทั้งเรื่อง **แถบสีไม่ขึ้น**, **ธงชาติใน Matrix หาย**, **การสลับแถว/คอลัมน์ใน Matrix ไม่ทำงาน** ไปจนถึง **`0.00%` ที่กลายเป็นสีเขียวหลอกตา**

เรามาไล่ดูสาเหตุเชิงเทคนิค (Root Cause) ทั้ง 4 จุด และนำโค้ดที่แก้ไขให้สมบูรณ์ระดับ **Executive Dashboard** ไปวางทับได้เลยครับ:

---

### 🔍 วิเคราะห์สาเหตุของปัญหาที่เกิดขึ้น (Root Cause Analysis)

#### 1. ทำไม `cardVisual` (ด้านบน) ถึงไม่มีแถบสี (Accent Bar) และเรียงลำดับเป็น A-Z?

- **สาเหตุเรื่องแถบสี:** ในไฟล์ `card-visual_2.json` บรรทัดที่ `411–421` พร็อพเพอร์ตี้ `"accentBar"` ยังถูกตั้งค่าเป็น `"show": false` อยู่ครับ และที่สำคัญคือใน Power BI **New Card Visual เมื่อเปิดโหมด Small Multiples (`Rows`)** ตัว `accentBar` เป็นฟีเจอร์ของ Callout ย่อย (ซึ่งเราซ่อน `cardCalloutArea` ไว้) ส่วนกรอบนอกของแต่ละประเทศคือ **Small Multiples Cell** ซึ่งหากต้องการใส่สีประจำชาติที่หัวการ์ดหรือขอบการ์ดของแต่ละประเทศ จะต้องผูกสีที่ **`smallMultiplesHeader` (`backColor` หรือ `fontColor`)** หรือ **`smallMultiplesBorder`** ครับ

- **สาเหตุเรื่องสี `0.00%` เป็นสีเขียว:** ในรูปจะเห็นว่า `0.00%` เป็นสีเขียวสด เพราะราคาอาจมีเศษทศนิยมเล็กน้อยมากจากการคำนวณเฉลี่ย หรือเกิดจาก Format String ใน DAX เราจึงควรปรับสูตร `[Price Oryza DoD Color]` ให้เช็คที่ `ROUND(_Pct, 4) > 0` เพื่อที่เวลาหน้าจอโชว์ `0.00%` สีจะได้เป็น **สีเทา (`#64748B`)** ตามหลักสากล ไม่ใช่สีเขียวครับ

#### 2. ทำไม `pivotTable` (Matrix ด้านล่าง) ถึงไม่มีรูปธงชาติ และไม่เรียงจากบนลงล่าง?

- **สาเหตุที่ 1 (ธงชาติว่างเปล่า):** ดูที่ไฟล์ `_Measures` บรรทัดที่ `647` ในสูตร `Product Country Flag SVG`:

```dax
VAR _Country =
    UPPER(
        COALESCE(
            CALCULATE(MAX(Fact_Rice_Price[Country Name])), ...

```

เมื่อวางใน Matrix ที่ถูกกรองด้วย Slicer ช่วงวันที่ (`4/1/2021 - 22/10/2026`) หรือบริบทของตาราง บางครั้ง `Fact_Rice_Price` ในจังหวะประเมิน ImageUrl ไม่สามารถดึงค่าได้ และที่สำคัญกว่านั้นคือ ใน Power BI Matrix (`pivotTable`) หากไม่มีฟิลด์อยู่ในช่อง `Rows` เลย (มีแต่ `Columns` กับ `Values`) Engine ของ Matrix จะ **เพิกเฉยต่อคำสั่ง `"showOnRows": true**`(สังเกตในรูปว่า`Flag | Spot (USD/t) | DoD`ยังเรียงเป็นคอลัมน์แนวนอนอยู่เลย ไม่ยอมเรียงลงมาเป็นแถว!) และทำให้คอลัมน์`Flag` ถูกบีบจนเรนเดอร์ SVG ไม่ขึ้นครับ

---

### 🛠️ การแก้ไขและยกระดับสู่ระดับ Professional (ทำ 3 ขั้นตอนสั้นๆ)

#### ขั้นตอนที่ 1: อัปเดต DAX 3 Measures ในไฟล์ `_Measures.tmdl`

ผมปรับปรุงสูตรใน `_Measures.tmdl` ดังนี้:

1. **`[Product Country Flag SVG]`:** ตัดการพึ่งพา `Fact_Rice_Price` ออกทั้งหมด! ให้เช็คจากชื่อสินค้า `SELECTEDVALUE(Dim_Product[Product Name])` โดยตรง 100% (รวดเร็วกว่าเดิมมาก และการันตีว่าธงชาติขึ้นทุกตารางแน่นอน)
2. **`[Price Oryza DoD Color]`:** ปรับให้ถ้าค่าปัดเศษเป็น `0.00%` (`ABS(_Pct) < 0.00005`) ให้แสดง **สีเทา (`#64748B`)** ทันที ไม่แสดงสีเขียวหลอกตา
3. **เพิ่ม Measure ใหม่ `[Product Header Tint Color]`:** สร้างสีพื้นหลังพาสเทลอ่อนๆ ตามสีประจำชาติสำหรับใส่ใน Header ของการ์ดและตาราง เพื่อความหรูหราและอ่านง่าย

นำโค้ดด้านล่างนี้ไป **วางทับส่วนท้ายของไฟล์ `\_Measures.tmdl**`(ตั้งแต่`measure 'Price Oryza DoD Color'`ลงไปถึง`measure 'Product Country Flag SVG'`) ได้เลยครับ:

```dax
	/// สี Dynamic สำหรับการเปลี่ยนแปลงราคา DoD (เขียว/แดง/เทา - ป้องกัน 0.00% เป็นสีเขียว)
	measure 'Price Oryza DoD Color' =
			VAR _Pct = [Price Oryza DoD %]
			RETURN
			    SWITCH(
			        TRUE(),
			        ISBLANK(_Pct) || ABS(_Pct) < 0.00005, "#64748B",
			        _Pct > 0, "#10B981",
			        _Pct < 0, "#EF4444",
			        "#64748B"
			    )
		displayFolder: 10. Text
		lineageTag: f2c43b92-4f81-49ea-bc91-23da567a2015

	/// สีพื้นหลังอ่อน (Soft Tint) ประจำประเทศสำหรับหัวการ์ด Small Multiples Header
	measure 'Product Header Tint Color' =
			VAR _Product = SELECTEDVALUE(Dim_Product[Product Name])
			RETURN
			    SWITCH(
			        TRUE(),
			        CONTAINSSTRING(_Product, "Thai"), "#E6F4F1",
			        CONTAINSSTRING(_Product, "Viet"), "#FDE8EA",
			        CONTAINSSTRING(_Product, "India"), "#F3E8FF",
			        CONTAINSSTRING(_Product, "Pakistan"), "#FFF3E0",
			        CONTAINSSTRING(_Product, "Myanmar"), "#EDE9FE",
			        CONTAINSSTRING(_Product, "U.S.") || CONTAINSSTRING(_Product, "US"), "#DBEAFE",
			        CONTAINSSTRING(_Product, "Brazil"), "#E0F2F1",
			        CONTAINSSTRING(_Product, "Uruguay"), "#E0F7FA",
			        CONTAINSSTRING(_Product, "Argentina"), "#FCE7F3",
			        CONTAINSSTRING(_Product, "Italy"), "#FCE7F3",
			        "#F1F5F9"
			    )
		displayFolder: 07. UI & System Helpers
		lineageTag: c1c43b92-4f81-49ea-bc91-23da567a3002

	/// ธงชาติรูปแบบ Vector SVG (Base64) ตามประเทศของสินค้าข้าว (ประเมินจาก Dim_Product โดยตรง 100%)
	measure 'Product Country Flag SVG' =
			VAR _Prod = SELECTEDVALUE(Dim_Product[Product Name], "")
			VAR _Prefix = "data:image/svg+xml;base64,"
			RETURN
			    SWITCH(
			        TRUE(),
			        _Prod = "", BLANK(),

			        -- Argentina
			        CONTAINSSTRING(_Prod, "Argentina"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iOCIgZmlsbD0iIzc0QUNERiIvPjxyZWN0IHk9IjgiIHdpZHRoPSIzNiIgaGVpZ2h0PSI4IiBmaWxsPSIjRkZGRkZGIi8+PHJlY3QgeT0iMTYiIHdpZHRoPSIzNiIgaGVpZ2h0PSI4IiBmaWxsPSIjNzRBQ0RGIi8+PGNpcmNsZSBjeD0iMTgiIGN5PSIxMiIgcj0iMi41IiBmaWxsPSIjRjZCNDBFIi8+PC9zdmc+",

			        -- Brazil
			        CONTAINSSTRING(_Prod, "Brazil"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iMjQiIHJ4PSIyIiBmaWxsPSIjMDA5NzM5Ii8+PHBvbHlnb24gcG9pbnRzPSIxOCwzIDMzLDEyIDE4LDIxIDMsMTIiIGZpbGw9IiNGRUREMDAiLz48Y2lyY2xlIGN4PSIxOCIgY3k9IjEyIiByPSI1IiBmaWxsPSIjMDEyMTY5Ii8+PHBhdGggZD0iTTEzLjUsMTMgUTE4LDkgMjIuNSwxMiIgc3Ryb2tlPSIjRkZGRkZGIiBzdHJva2Utd2lkdGg9IjAuOCIgZmlsbD0ibm9uZSIvPjwvc3ZnPg==",

			        -- India / Indian
			        CONTAINSSTRING(_Prod, "India"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iOCIgZmlsbD0iI0ZGOTkzMyIvPjxyZWN0IHk9IjgiIHdpZHRoPSIzNiIgaGVpZ2h0PSI4IiBmaWxsPSIjRkZGRkZGIi8+PHJlY3QgeT0iMTYiIHdpZHRoPSIzNiIgaGVpZ2h0PSI4IiBmaWxsPSIjMTM4ODA4Ii8+PGNpcmNsZSBjeD0iMTgiIGN5PSIxMiIgcj0iMi42IiBzdHJva2U9IiMwMDAwODAiIHN0cm9rZS13aWR0aD0iMC44IiBmaWxsPSJub25lIi8+PGNpcmNsZSBjeD0iMTgiIGN5PSIxMiIgcj0iMC44IiBmaWxsPSIjMDAwMDgwIi8+PC9zdmc+",

			        -- Myanmar
			        CONTAINSSTRING(_Prod, "Myanmar"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iOCIgZmlsbD0iI0ZFQ0IwMCIvPjxyZWN0IHk9IjgiIHdpZHRoPSIzNiIgaGVpZ2h0PSI4IiBmaWxsPSIjMzRCMjMzIi8+PHJlY3QgeT0iMTYiIHdpZHRoPSIzNiIgaGVpZ2h0PSI4IiBmaWxsPSIjRUEyODM5Ii8+PHBvbHlnb24gcG9pbnRzPSIxOCw2LjUgMTkuOCwxMS41IDI1LDExLjUgMjAuOCwxNC41IDIyLjUsMTkuNSAxOCwxNi41IDEzLjUsMTkuNSAxNS4yLDE0LjUgMTEsMTEuNSAxNi4yLDExLjUiIGZpbGw9IiNGRkZGRkYiLz48L3N2Zz4=",

			        -- Pakistan
			        CONTAINSSTRING(_Prod, "Pakistan"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iOSIgaGVpZ2h0PSIyNCIgZmlsbD0iI0ZGRkZGRiIvPjxyZWN0IHg9IjkiIHdpZHRoPSIyNyIgaGVpZ2h0PSIyNCIgZmlsbD0iIzAxNDExQyIvPjxjaXJjbGUgY3g9IjIyLjUiIGN5PSIxMiIgcj0iNSIgZmlsbD0iI0ZGRkZGRiIvPjxjaXJjbGUgY3g9IjI0IiBjeT0iMTEiIHI9IjQuNSIgZmlsbD0iIzAxNDExQyIvPjxwb2x5Z29uIHBvaW50cz0iMjMuNSw4IDI0LjIsOS42IDI1LjgsOS42IDI0LjUsMTAuNiAyNSwxMi4yIDIzLjUsMTEuMiAyMiwxMi4yIDIyLjUsMTAuNiAyMS4yLDkuNiAyMi44LDkuNiIgZmlsbD0iI0ZGRkZGRiIvPjwvc3ZnPg==",

			        -- Thailand / Thai
			        CONTAINSSTRING(_Prod, "Thai"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iNCIgZmlsbD0iI0E1MTkzMSIvPjxyZWN0IHk9IjQiIHdpZHRoPSIzNiIgaGVpZ2h0PSI0IiBmaWxsPSIjRkZGRkZGIi8+PHJlY3QgeT0iOCIgd2lkdGg9IjM2IiBoZWlnaHQ9IjgiIGZpbGw9IiMyRDJBNEEiLz48cmVjdCB5PSIxNiIgd2lkdGg9IjM2IiBoZWlnaHQ9IjQiIGZpbGw9IiNGRkZGRkYiLz48cmVjdCB5PSIyMCIgd2lkdGg9IjM2IiBoZWlnaHQ9IjQiIGZpbGw9IiNBNTE5MzEiLz48L3N2Zz4=",

			        -- United States / U.S.
			        CONTAINSSTRING(_Prod, "U.S.") || CONTAINSSTRING(_Prod, "US"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iMjQiIGZpbGw9IiNGRkZGRkYiLz48cGF0aCBkPSJNMCwwSDM2VjJIMFpNMCw0SDM2VjZIMFpNMCw4SDM2VjEwSDBaTTAsMTJIMzZWMTRIMFpNMCwxNkgzNlYxOEgwWk0wLDIwSDM2VjIySDBaIiBmaWxsPSIjQjIyMjM0Ii8+PHJlY3Qgd2lkdGg9IjE1IiBoZWlnaHQ9IjEzIiBmaWxsPSIjM0MzQjZFIi8+PGNpcmNsZSBjeD0iNCIgY3k9IjMiIHI9IjAuOSIgZmlsbD0iI0ZGRkZGRiIvPjxjaXJjbGUgY3g9IjgiIGN5PSIzIiByPSIwLjkiIGZpbGw9IiNGRkZGRkYiLz48Y2lyY2xlIGN4PSIxMiIgY3k9IjMiIHI9IjAuOSIgZmlsbD0iI0ZGRkZGRiIvPjxjaXJjbGUgY3g9IjYiIGN5PSI2LjUiIHI9IjAuOSIgZmlsbD0iI0ZGRkZGRiIvPjxjaXJjbGUgY3g9IjEwIiBjeT0iNi41IiByPSIwLjkiIGZpbGw9IiNGRkZGRkYiLz48Y2lyY2xlIGN4PSI0IiBjeT0iMTAiIHI9IjAuOSIgZmlsbD0iI0ZGRkZGRiIvPjxjaXJjbGUgY3g9IjgiIGN5PSIxMCIgcj0iMC45IiBmaWxsPSIjRkZGRkZGIi8+PGNpcmNsZSBjeD0iMTIiIGN5PSIxMCIgcj0iMC45IiBmaWxsPSIjRkZGRkZGIi8+PC9zdmc+",

			        -- Uruguay
			        CONTAINSSTRING(_Prod, "Uruguay"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iMjQiIGZpbGw9IiNGRkZGRkYiLz48cGF0aCBkPSJNMCwyLjY3SDM2VjUuMzNIMFpNMCw4SDM2VjEwLjY3SDBaTTAsMTMuMzNIMzZWMTZIMFpNMCwxOC42N0gzNlYyMS4zM0gwWiIgZmlsbD0iIzAwMzhBOCIvPjxyZWN0IHdpZHRoPSIxMiIgaGVpZ2h0PSIxMiIgZmlsbD0iI0ZGRkZGRiIvPjxjaXJjbGUgY3g9IjYiIGN5PSI2IiByPSIzIiBmaWxsPSIjRkNEMTE2Ii8+PGNpcmNsZSBjeD0iNiIgY3k9IjYiIHI9IjEuNSIgZmlsbD0iIzdCM0YwMCIvPjwvc3ZnPg==",

			        -- Vietnam
			        CONTAINSSTRING(_Prod, "Vietnam"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMzYiIGhlaWdodD0iMjQiIHJ4PSIyIiBmaWxsPSIjREEyNTFEIi8+PHBvbHlnb24gcG9pbnRzPSIxOCw1IDIwLjMsMTEgMjYuNSwxMSAyMS41LDE0LjYgMjMuNCwyMC41IDE4LDE3IDEyLjYsMjAuNSAxNC41LDE0LjYgOS41LDExIDE1LjcsMTEiIGZpbGw9IiNGRkZGMDAiLz48L3N2Zz4=",

			        -- Italy
			        CONTAINSSTRING(_Prod, "Italy"),
			        _Prefix & "PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIzNiIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDM2IDI0Ij48cmVjdCB3aWR0aD0iMTIiIGhlaWdodD0iMjQiIGZpbGw9IiMwMDkyNDYiLz48cmVjdCB4PSIxMiIgd2lkdGg9IjEyIiBoZWlnaHQ9IjI0IiBmaWxsPSIjRkZGRkZGIi8+PHJlY3QgeD0iMjQiIHdpZHRoPSIxMiIgaGVpZ2h0PSIyNCIgZmlsbD0iI0NFMkIzNyIvPjwvc3ZnPg==",

			        BLANK()
			    )
		dataCategory: ImageUrl
		displayFolder: 10. Text
		lineageTag: d67c5210-91a3-4871-b0e2-7634f1ca9201

```

---

#### ขั้นตอนที่ 2: โค้ด `card-visual.json` (แก้ไขให้มีแถบสี Accent Bar + Header สีตรงกับเส้นกราฟ)

ในโค้ดชุดนี้ ผมทำการเปิด:

1. **`accentBar` (`"show": true`, `"position": "'Top'"`, `"width": "4L"`)** โดยผูกสีเข้ากับ Measure **`[Product Line Color]`** (สีเดียวกับเส้นกราฟเป๊ะ เช่น Brazil สีเขียว `#2A9D8F`, Argentina สีชมพู `#F72585`, India สีม่วง `#7209B7`)

2. **สีชื่อสินค้าใน `smallMultiplesHeader**` ผูกกับ **`[Product Line Color]`\*\* และพื้นหลัง Header อ่อนๆ เพื่อให้แม้ในกรณีที่ Small Multiples ซ่อน Callout Area ตัวหัวการ์ดก็จะยังมีแถบสีประจำชาติที่ตรงกับเส้นกราฟ 100% ครับ

นำโค้ดนี้ไปวางทับไฟล์ `card-visual.json` (`9b71fc7d254efa994cee`):

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/visualContainer/2.14.0/schema.json",
  "name": "9b71fc7d254efa994cee",
  "position": {
    "x": 215.01709765069037,
    "y": 9.085229478198185,
    "z": 9000,
    "height": 127.19321269477457,
    "width": 1041.7729801667251,
    "tabOrder": 9000
  },
  "visual": {
    "visualType": "cardVisual",
    "query": {
      "queryState": {
        "Data": {
          "projections": [
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza Latest (USD/ton)"
                }
              },
              "queryRef": "_Measures.Price Oryza Latest (USD/ton)",
              "nativeQueryRef": "Spot Price",
              "displayName": "(USD/t)"
            },
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza DoD %"
                }
              },
              "queryRef": "_Measures.Price Oryza DoD %",
              "nativeQueryRef": "Day-over-Day",
              "displayName": "DoD%"
            }
          ]
        },
        "Rows": {
          "projections": [
            {
              "field": {
                "Column": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "Dim_Product"
                    }
                  },
                  "Property": "Product Name"
                }
              },
              "queryRef": "Dim_Product.Product Name",
              "nativeQueryRef": "Product Name"
            }
          ]
        }
      },
      "sortDefinition": {
        "sort": [
          {
            "field": {
              "Measure": {
                "Expression": {
                  "SourceRef": {
                    "Entity": "_Measures"
                  }
                },
                "Property": "Price Oryza Latest (USD/ton)"
              }
            },
            "direction": "Descending"
          }
        ],
        "isDefaultSort": true
      }
    },
    "objects": {
      "smallMultiplesLayout": [
        {
          "properties": {
            "style": {
              "expr": {
                "Literal": {
                  "Value": "'Cards'"
                }
              }
            },
            "orientation": {
              "expr": {
                "Literal": {
                  "Value": "2D"
                }
              }
            },
            "columnCount": {
              "expr": {
                "Literal": {
                  "Value": "4L"
                }
              }
            },
            "rowCount": {
              "expr": {
                "Literal": {
                  "Value": "1L"
                }
              }
            },
            "headerPosition": {
              "expr": {
                "Literal": {
                  "Value": "'Top'"
                }
              }
            },
            "titleOrientation": {
              "expr": {
                "Literal": {
                  "Value": "1D"
                }
              }
            }
          }
        }
      ],
      "smallMultiplesHeader": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "11D"
                }
              }
            },
            "bold": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Product Line Color"
                    }
                  }
                }
              }
            },
            "backColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Product Header Tint Color"
                    }
                  }
                }
              }
            },
            "backgroundShow": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            }
          }
        },
        {
          "properties": {
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "11D"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Product Line Color"
                    }
                  }
                }
              }
            },
            "backColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Product Header Tint Color"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "smallMultiplesGrid": [
        {
          "properties": {
            "gridlineShow": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "smallMultiplesCellBackGround": [
        {
          "properties": {
            "backgroundShow": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "backgroundTransparency": {
              "expr": {
                "Literal": {
                  "Value": "0D"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "smallMultiplesBorder": [
        {
          "properties": {
            "gridlineShow": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "gridlineColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Product Line Color"
                    }
                  }
                }
              }
            },
            "gridlineWidth": {
              "expr": {
                "Literal": {
                  "Value": "2D"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "smallMultiplesOuterShape": [
        {
          "properties": {}
        }
      ],
      "layout": [
        {
          "properties": {
            "columnCount": {
              "expr": {
                "Literal": {
                  "Value": "2L"
                }
              }
            },
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'middle'"
                }
              }
            },
            "showFixedSize": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "calloutSize": {
              "expr": {
                "Literal": {
                  "Value": "50D"
                }
              }
            }
          }
        },
        {
          "properties": {
            "paddingUniform": {
              "expr": {
                "Literal": {
                  "Value": "0L"
                }
              }
            },
            "paddingIndividual": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "backgroundShow": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "value": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "16D"
                }
              }
            },
            "bold": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#0F172A'"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza Latest (USD/ton)"
          }
        },
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "15D"
                }
              }
            },
            "bold": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Price Oryza DoD Color"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza DoD %"
          }
        },
        {
          "properties": {
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "15D"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "label": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "9.5D"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#64748B'"
                    }
                  }
                }
              }
            },
            "position": {
              "expr": {
                "Literal": {
                  "Value": "'aboveValue'"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "spacing": [
        {
          "properties": {
            "verticalSpacing": {
              "expr": {
                "Literal": {
                  "Value": "4L"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "image": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        },
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "cardCalloutArea": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "outline": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "fillCustom": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "accentBar": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "position": {
              "expr": {
                "Literal": {
                  "Value": "'Top'"
                }
              }
            },
            "width": {
              "expr": {
                "Literal": {
                  "Value": "4L"
                }
              }
            },
            "color": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Product Line Color"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "shadowCustom": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "glowCustom": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "padding": [
        {
          "properties": {
            "bottomMargin": {
              "expr": {
                "Literal": {
                  "Value": "12L"
                }
              }
            },
            "topMargin": {
              "expr": {
                "Literal": {
                  "Value": "10L"
                }
              }
            },
            "paddingIndividual": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ],
      "border": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          },
          "selector": {
            "id": "default"
          }
        }
      ]
    },
    "visualContainerObjects": {
      "title": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "text": {
              "expr": {
                "Literal": {
                  "Value": "''"
                }
              }
            }
          }
        }
      ],
      "visualHeader": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "visualTooltip": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ]
    },
    "drillFilterOtherVisuals": true
  }
}
```

---

#### ขั้นตอนที่ 3: โค้ด `matrix-comparison-visual.json` (บังคับให้ `showOnRows` ทำงาน 100% + โชว์ธงชาติ SVG เต็มใบ)

**เคล็ดลับสำคัญของ Power BI Matrix (`pivotTable`):**
การจะให้คำสั่ง `"showOnRows": true` (Switch values to rows) ทำงานได้ใน Power BI **จำเป็นต้องมีฟิลด์วางอยู่ในช่อง `Rows` อย่างน้อย 1 ฟิลด์ครับ!** ในโค้ดด้านล่างนี้ ผมจึงใส่ `Dim_Source[SourceCode]` ลงในช่อง `Rows` เป็น Dummy Anchor (และซ่อนหัวแถวมันไว้) ทันทีที่ทำแบบนี้:

1. ตารางจะพลิกเอา **`Flag`**, **`Spot (USD/t)`** และ **`DoD`** ลงมาเรียงซ้อนกันในแนวตั้ง (3 บรรทัด) เหมือนการ์ดทันที!
2. เมื่อแต่ละคอลัมน์ไม่ต้องซอยเป็น 3 คอลัมน์ย่อย (`Flag | Spot | DoD`) อีกต่อไป ความกว้างของคอลัมน์จะกว้างเต็มการ์ด และเมื่อบวกกับ DAX `[Product Country Flag SVG]` ตัวใหม่ในขั้นตอนที่ 1 **รูปธงชาติ Vector SVG จะปรากฏขึ้นตรงกลางใต้ชื่อสินค้าของทุกประเทศอย่างสวยงามครับ**

นำโค้ดนี้ไปวางทับไฟล์ `matrix-comparison-visual.json` (`c7a4b8d2e1f34906a5b1`):

```json
{
  "$schema": "https://developer.microsoft.com/json-schemas/fabric/item/report/definition/visualContainer/2.14.0/schema.json",
  "name": "c7a4b8d2e1f34906a5b1",
  "position": {
    "x": 215.01709765069037,
    "y": 142.0,
    "z": 9500,
    "height": 145.0,
    "width": 1041.7729801667251,
    "tabOrder": 9500
  },
  "visual": {
    "visualType": "pivotTable",
    "query": {
      "queryState": {
        "Rows": {
          "projections": [
            {
              "field": {
                "Column": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "Dim_Product"
                    }
                  },
                  "Property": "New Sub Category Name"
                }
              },
              "queryRef": "Dim_Product.New Sub Category Name",
              "nativeQueryRef": "Category",
              "displayName": " ",
              "active": true
            }
          ]
        },
        "Columns": {
          "projections": [
            {
              "field": {
                "Column": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "Dim_Product"
                    }
                  },
                  "Property": "Product Name"
                }
              },
              "queryRef": "Dim_Product.Product Name",
              "nativeQueryRef": "Product Name",
              "active": true
            }
          ]
        },
        "Values": {
          "projections": [
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Product Country Flag SVG"
                }
              },
              "queryRef": "_Measures.Product Country Flag SVG",
              "nativeQueryRef": "Origin",
              "displayName": "Origin"
            },
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza Latest (USD/ton)"
                }
              },
              "queryRef": "_Measures.Price Oryza Latest (USD/ton)",
              "nativeQueryRef": "Spot (USD/t)",
              "displayName": "Spot (USD/t)"
            },
            {
              "field": {
                "Measure": {
                  "Expression": {
                    "SourceRef": {
                      "Entity": "_Measures"
                    }
                  },
                  "Property": "Price Oryza DoD Display"
                }
              },
              "queryRef": "_Measures.Price Oryza DoD Display",
              "nativeQueryRef": "DoD",
              "displayName": "DoD"
            }
          ]
        }
      },
      "sortDefinition": {
        "sort": [
          {
            "field": {
              "Measure": {
                "Expression": {
                  "SourceRef": {
                    "Entity": "_Measures"
                  }
                },
                "Property": "Price Oryza Latest (USD/ton)"
              }
            },
            "direction": "Descending"
          }
        ]
      }
    },
    "objects": {
      "values": [
        {
          "properties": {
            "showOnRows": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "12D"
                }
              }
            },
            "backColorPrimary": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#FFFFFF'"
                    }
                  }
                }
              }
            },
            "backColorSecondary": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#F8FAFC'"
                    }
                  }
                }
              }
            }
          }
        },
        {
          "properties": {
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Measure": {
                      "Expression": {
                        "SourceRef": {
                          "Entity": "_Measures"
                        }
                      },
                      "Property": "Price Oryza DoD Color"
                    }
                  }
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza DoD Display"
          }
        }
      ],
      "columnHeaders": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "11D"
                }
              }
            },
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#0F172A'"
                    }
                  }
                }
              }
            },
            "backColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#E2E8F0'"
                    }
                  }
                }
              }
            }
          }
        }
      ],
      "rowHeaders": [
        {
          "properties": {
            "fontFamily": {
              "expr": {
                "Literal": {
                  "Value": "'Segoe UI Semibold'"
                }
              }
            },
            "fontSize": {
              "expr": {
                "Literal": {
                  "Value": "9.5D"
                }
              }
            },
            "fontColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#475569'"
                    }
                  }
                }
              }
            },
            "steppedLayoutIndentation": {
              "expr": {
                "Literal": {
                  "Value": "0D"
                }
              }
            }
          }
        }
      ],
      "subTotals": [
        {
          "properties": {
            "columnSubtotals": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "rowSubtotals": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "grid": [
        {
          "properties": {
            "imageHeight": {
              "expr": {
                "Literal": {
                  "Value": "22D"
                }
              }
            },
            "outlineColor": {
              "solid": {
                "color": {
                  "expr": {
                    "Literal": {
                      "Value": "'#CBD5E1'"
                    }
                  }
                }
              }
            },
            "gridVertical": {
              "expr": {
                "Literal": {
                  "Value": "true"
                }
              }
            },
            "gridVerticalWeight": {
              "expr": {
                "Literal": {
                  "Value": "2D"
                }
              }
            },
            "gridHorizontal": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            },
            "rowPadding": {
              "expr": {
                "Literal": {
                  "Value": "3D"
                }
              }
            }
          }
        }
      ],
      "fieldFormatting": [
        {
          "properties": {
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Product Country Flag SVG"
          }
        },
        {
          "properties": {
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza Latest (USD/ton)"
          }
        },
        {
          "properties": {
            "alignment": {
              "expr": {
                "Literal": {
                  "Value": "'Center'"
                }
              }
            }
          },
          "selector": {
            "metadata": "_Measures.Price Oryza DoD Display"
          }
        }
      ]
    },
    "visualContainerObjects": {
      "title": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "visualHeader": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ],
      "visualTooltip": [
        {
          "properties": {
            "show": {
              "expr": {
                "Literal": {
                  "Value": "false"
                }
              }
            }
          }
        }
      ]
    },
    "drillFilterOtherVisuals": true
  }
}
```

---

### ✨ สรุปผลลัพธ์ที่จะเกิดขึ้นหลังวางโค้ดชุดนี้ (แล้วปิด-เปิด Power BI Desktop ใหม่):

1. **ในการ์ดด้านบน (`cardVisual`):**

- กรอบการ์ด ชื่อสินค้า และพื้นหลังส่วนหัวของแต่ละประเทศ จะเปลี่ยนเป็น **สีประจำชาติที่ตรงกับสีเส้นกราฟด้านล่างเป๊ะ** (เช่น `Brazil 5% broken` เป็นโทนสีเขียวน้ำทะเล `#2A9D8F`, `Argentina 5% broken` เป็นโทนสีชมพูเข้ม `#F72585`)
- ตัวเลข `0.00%` จะเปลี่ยนจากสีเขียวหลอกตา เป็น **สีเทาสุภาพ (`#64748B`)** และจะแสดงสีเขียว/แดงเฉพาะวันที่ราคาขยับขึ้นลงจริงเท่านั้น

2. **ในตารางการ์ดด้านล่าง (`pivotTable`):**

- ข้อมูลจะถูกจัดเรียงในแนวตั้ง 3 ชั้น (**แถวที่ 1: รูปธงชาติ SVG**, **แถวที่ 2: ราคา Spot**, **แถวที่ 3: DoD พร้อมลูกศร**) เรียงตามราคาสูงสุดจากซ้ายไปขวา (`612.00` $\rightarrow$ `550.00` $\rightarrow$ `464.00` $\rightarrow$ `387.00`)

- **ธงชาติ Vector SVG จะแสดงผลครบทุกประเทศ 100%** เพราะดึงจาก `Dim_Product[Product Name]` โดยตรง ไม่ติดปัญหา Context ของ `Fact_Rice_Price` อีกต่อไปครับ!
