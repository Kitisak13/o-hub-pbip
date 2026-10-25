# Power BI Visual Documentation: Oryza Spot Price Card Visual

เอกสารรวบรวมไฟล์สำคัญ, สถาปัตยกรรม, ปัญหาที่พบในการพัฒนา (Troubleshooting & Root Cause Analysis) และแนวทางแก้ไขเรื่องการแสดงผล **National Flag (ธงชาติ)** บน New Card Visual ในหน้า **Oryza** (`pages/76e5dee052e5452704ca`)

---

## 1. ข้อมูลและโครงสร้างไฟล์ในโฟลเดอร์นี้

| ชื่อไฟล์ | ที่มาเดิม | หน้าที่ |
| :--- | :--- | :--- |
| `card-visual.json` | `bi-rice-price.Report/.../9b71fc7d254efa994cee/visual.json` | โครงสร้าง PBIR Visual Container ของ New Card Visual (`cardVisual`) |
| `oryza-page.json` | `bi-rice-price.Report/.../76e5dee052e5452704ca/page.json` | นิยามของหน้ารายงาน "Oryza" (ขนาดหน้าจอ, Layout, Filter ประจำหน้า) |
| `Measures_Definition.tmdl` | `bi-rice-price.SemanticModel/.../_Measures.tmdl` | โค้ด DAX Measures ทั้งหมด รวมถึง `[Price Oryza Latest (USD/ton)]`, `[Price Oryza DoD %]` และ `[Product Country Flag SVG]` |

---

## 2. วัตถุประสงค์และการออกแบบของ Visual

* **Visual Type:** Microsoft Power BI **New Card Visual** (`cardVisual`)
* **Data Metrics (Callout Values):**
  1. `Spot Price`: ราคาข้าวล่าสุด `[Price Oryza Latest (USD/ton)]` (เช่น `550.00`)
  2. `Day-over-Day`: อัตราการเปลี่ยนแปลงราคารายวัน `[Price Oryza DoD %]` (เช่น `0.00%` พร้อม Format สีเขียว/แดง)
* **การจัดกลุ่มย่อย (Small Multiples Grid):**
  - ใช้ฟิลด์ `Dim_Product[Product Name]` วางในช่อง **Rows** เพื่อสร้างตารางการ์ด 4 คอลัมน์ x 1 แถว (`columnCount: 4L`, `rowCount: 1L`)
  - แสดงสินค้าข้าว Oryza ตัวท็อป 4 รายการ ได้แก่:
    1. Argentina 5% broken
    2. Brazil 5% broken
    3. India 5% broken
    4. Myanmar 5% broken

---

## 3. ปัญหาที่พบระหว่างการพัฒนาและสาเหตุเชิงลึก (Root Cause Analysis)

### ปัญหาที่ 1: ธงชาติแสดงเป็นกล่องสี่เหลี่ยมสีดำทึบ (Solid Black Flag)
* **อาการ:** เมื่อเพิ่มรูปภาพธงชาติเข้าไปในการ์ดครั้งแรก พบว่าธงชาติของทุกประเทศกลายเป็นรูปสี่เหลี่ยมผืนผ้าสีดำทึบทั้งหมด
* **สาเหตุที่แท้จริง:**
  - เดิมโค้ด DAX ส่งข้อมูล SVG ผ่าน Data URI รูปแบบ `data:image/svg+xml;utf8,...` และใช้การเข้ารหัสสีแบบ URL-encoded hex เช่น `<rect fill='%23FECB00'/>`
  - เอนจินเรนเดอร์ Chromium / WebView2 ภายใน Power BI Desktop ไม่สามารถ Parse ค่า `%23` ในแท็ก SVG ได้ถูกต้อง จึงเกิด Error การตีความค่าสีใน CSS และตกไปใช้ค่าสีเริ่มต้นตามมาตรฐาน SVG Spec (Fallback to initial value) ซึ่งก็คือ **สีดำ (`#000000`)**
* **แนวทางแก้ไขที่สำเร็จ:**
  - ปรับการสร้าง SVG ใน DAX ให้ส่งข้อมูลแบบ **Base64 Data URI** (`data:image/svg+xml;base64,...`)
  - โค้ดสี `#HEX` มาตรฐานจะถูกฝังอยู่ในระดับ Binary Payload ทำให้ข้ามปัญหา URL Encoding ทั้งหมด และแสดงสีจริงของธงชาติได้คมชัดทุกสี

---

### ปัญหาที่ 2: ข้อความและตัวเลขถูกบีบอัดจนขึ้น `...` (`Spot...` และ `5...`)
* **อาการ:** เมื่อมีรูปภาพธงชาติปรากฏ ข้อความคำว่า `Spot Price` และตัวเลขราคาถูกตัดทอนกลายเป็น `Spot...` และ `5...`
* **สาเหตุที่แท้จริง:**
  - Card Tile 1 ใบ มีความกว้างประมาณ ~255px และแบ่ง layout ด้านในออกเป็น 2 คอลัมน์ (คอลัมน์ละ ~120px)
  - เดิมรูปภาพถูกกำหนดขนาดไว้ที่ `28L` (28px / 28%) พร้อม Padding `6L` และฟอนต์ตัวเลขตั้งไว้ที่ `24pt` ทำให้พื้นที่แนวนอนในคอลัมน์แรกไม่เพียงพอ
* **แนวทางแก้ไขที่สำเร็จ:**
  - ปรับขนาด Flag Image เหลือ **`16L`** (สัดส่วน ~24x16px) และ Padding เหลือ **`4L`**
  - ปรับขนาดฟอนต์ตัวเลข Spot Price เป็น **`15pt`** (Segoe UI Semibold)
  - ปรับขนาดฟอนต์คำอธิบาย Label เป็น **`9.5pt`**
  - ผลลัพธ์: ข้อความ `Spot Price` และตัวเลขราคา เช่น `550.00` แสดงผลได้เต็มคำ ไม่ถูกตัดทอน

---

### ปัญหาที่ 3: ธงชาติแสดงเป็นธงชาติ Vietnam สำหรับการ์ดสินค้าทุกใบ
* **อาการ:** หลังจากแก้ไข Base64 ธงชาติเริ่มแสดงสีจริง แต่พบว่าการ์ดของ Argentina, Brazil, India กลับแสดงเป็น **ธงชาติเวียดนาม (พื้นแดงดาวเหลือง)** เหมือนกันหมด
* **สาเหตุเชิงลึก (Power BI Visual Engine Limitation):**
  - ใน New Card Visual (`cardVisual`) เมื่อใช้ **Small Multiples** (วางสินค้าในช่อง `Rows`) คุณสมบัติ `objects.image.imageUrl` ถูกผูกไว้กับระดับ **Callout Value ของ Visual รวม** ไม่ได้ถูกแยกคำนวณตามแถวของ Small Multiples
  - Power BI Desktop ส่งคำสั่ง DAX Query ดังนี้:
    ```dax
    EVALUATE
        SUMMARIZECOLUMNS(
            __DS0FilterTable,
            __DS0FilterTable2,
            "Product_Country_Flag_SVG", IGNORE('_Measures'[Product Country Flag SVG])
        )
    ```
  - สังเกตว่าใน Query การประเมิน `Product_Country_Flag_SVG` **ไม่มี `'Dim_Product'[Product Name]` อยู่ในการจัดกลุ่มแถวเลย**
  - เมื่อไม่มี Filter Context ของสินค้าในการคำนวณระดับบนสุด ฟังก์ชัน `_Country = CALCULATE(MAX(Fact_Rice_Price[Country Name]))` ใน DAX จึงคืนค่าชื่อประเทศที่มีลำดับอักษรสูงสุดในตาราง ซึ่งก็คือ **"VIETNAM"**
  - ส่งผลให้ DAX คืนค่าธงเวียดนามเพียงภาพเดียว และ Power BI นำภาพธงเวียดนามนั้นไปแปะให้กับการ์ดย่อยทุกใบใน Visual เท่ากันทั้งหมด

---

### ปัญหาที่ 4: การย้ายไอคอนธงชาติไปไว้หน้าชื่อสินค้าด้านบน (`smallMultiplesHeader`)
* **ความต้องการ:** ย้ายธงชาติไปไว้หน้าชื่อสินค้าส่วนบนของการ์ด เช่น `[flag] Argentina 5% Broken`
* **ข้อจำกัดทางเทคนิค:**
  1. ส่วนหัวด้านบนของการ์ดคือ **Small Multiples Header** ซึ่ง Power BI ออกแบบมาให้รับเฉพาะ **ฟิลด์ข้อความ (Pure Text Column)** เท่านั้น ไม่รองรับการใส่ Image URL หรือเรนเดอร์ภาพกราฟิก SVG
  2. หากใช้ **Unicode Emoji ธงชาติ** (เช่น 🇦🇷, 🇧🇷, 🇮🇳, 🇹🇭):
     - บน **Windows (Power BI Desktop)**: ฟอนต์ระบบ `Segoe UI Emoji` ของ Microsoft **ไม่มีภาพกราฟิกธงชาติแบบสี** (เป็นนโยบายของ Microsoft) จึงแสดงผลเป็น **ตัวอักษรย่อขาวดำ 2 ตัว เช่น `AR`, `BR`, `IN`, `TH`**
     - บน **Mac / iPhone / Android / Power BI Service Web**: จะเรนเดอร์เป็นภาพธงชาติสีสันสดใสโดยอัตโนมัติ

---

## 4. ทางเลือกและแนวทางแก้ไขสำหรับให้ทีมงานพิจารณา (Solutions for Discussion)

### ทางเลือก A: เพิ่ม Country Code Badge ในชื่อสินค้า (แนะนำ ⭐)
* **วิธีทำ:** ปรับชื่อหัวการ์ดให้มีรหัสย่อประเทศกำกับ เช่น:
  - `[ARG] Argentina 5% broken`
  - `[BRA] Brazil 5% broken`
  - `[IND] India 5% broken`
  - `[MMR] Myanmar 5% broken`
* **การปรับแต่ง:** **ปิดรูปภาพ Image URL ด้านล่างออก**
* **ข้อดี:**
  - สวยงาม คลีน สะอาดตา เป็นมาตรฐานสากลตามแบบฉบับ Executive Dashboard
  - คืนพื้นที่แนวนอนให้ตัวเลข `Spot Price 550.00` และ `0.00%` แสดงผลได้โปร่งโล่ง ไม่แออัด
  - ทำงานได้อย่างเสถียร 100% บนทุกแพลตฟอร์ม (ทั้ง Desktop, Web, Mobile)

---

### ทางเลือก B: ใส่ Unicode Flag Emoji ในชื่อสินค้า
* **วิธีทำ:** ปรับชื่อหัวการ์ดเป็น `🇦🇷 Argentina 5% broken`, `🇧🇷 Brazil 5% broken`
* **การปรับแต่ง:** ปิดรูปภาพ Image URL ด้านล่างออก
* **ข้อสังเกต:**
  - บนเครื่อง Windows ของ Power BI Desktop จะแสดงผลเป็นตัวย่อกรอบสี่เหลี่ยม `AR`, `BR`
  - เมื่อ Publish ขึ้น Cloud และเปิดดูผ่าน Mac / iPad / มือถือ จะกลายเป็นธงชาติหลากสีสวยงาม

---

### ทางเลือก C: แยกเป็นการ์ดเดี่ยว 4 ใบ (Standalone Card Visuals)
* **วิธีทำ:** เปลี่ยนจากการใช้ Small Multiples 1 Visual เป็นการวาง **New Card Visual แยก 4 ใบอิสระ** เรียงแนวนอน
* **ข้อดี:**
  - การ์ดแต่ละใบจะมี Filter Context ของสินค้าและประเทศนั้นๆ แบบ 100%
  - สามารถแสดง **ภาพกราฟิกธงชาติ Vector SVG สีสันสดใส** แยกตามประเทศได้อย่างถูกต้อง 100% บนทุกอุปกรณ์
  - สามารถปรับแต่งย้ายตำแหน่งรูปธงชาติขึ้นไปอยู่ด้านบนของการ์ดได้อิสระ
* **ข้อพิจารณา:** การจัดเรียงต้องฟิกซ์ตำแหน่งสินค้า 4 ตัวหลัก (ไม่หมุนเวียนอัตโนมัติตาม Dynamic Sort)

---

### ทางเลือก D: ใช้ Card Matrix / Table Custom Layout
* **วิธีทำ:** ออกแบบเป็น Matrix หรือ Table ที่ซ่อนขอบตาราง (Borderless) โดยแบ่งคอลัมน์เป็น [Flag SVG] | [Product Name] | [Spot Price] | [DoD %]
* **ข้อดี:**
  - รองรับทั้ง Dynamic Sorting และการแสดงภาพ Vector SVG รายแถว/รายคอลัมน์ได้อย่างสมบูรณ์แบบ

---

## 5. การดำเนินการและบทสรุปจากทีมงาน (DA-review1 & DA-review2 Implementation)

หลังจากการปรึกษาทีมงานและรับข้อเสนอแนะจาก Data Analyst (ดูรายละเอียดใน `DA-review1.md` และ `DA-review2.md`):

### 1. New Card Visual (รูปแบบที่ 1 - `9b71fc7d254efa994cee`)
- **การออกแบบ:** ดีไซน์สไตล์คลีนระดับผู้บริหาร (Executive Command) ไม่มีไอคอนธงชาติมาบดบังตัวเลข
- **การแก้ปัญหาแถบสี (Accent Bar) & ขอบสีประจำชาติ:** 
  - ผูกสีตัวอักษรและขอบใน `smallMultiplesHeader` และ `smallMultiplesBorder` เข้ากับ Measure `[Product Line Color]` เพื่อให้สีตรงกับเส้นกราฟด้านล่าง 100%
  - ผูกสีพื้นหลังหัวการ์ดเข้ากับ Measure ใหม่ `[Product Header Tint Color]` เพื่อให้มีสีพาสเทลอ่อนๆ สบายตา
- **การแก้ปัญหา `0.00%` สีเขียวหลอกตา:** ปรับเงื่อนไขใน Measure `[Price Oryza DoD Color]` ให้ตรวจสอบ `ABS(_Pct) < 0.00005` บังคับให้เป็น **สีเทา (`#64748B`)** เมื่อไม่มีการเปลี่ยนแปลงราคา

### 2. Matrix Comparison Visual (รูปแบบที่ 2 - `c7a4b8d2e1f34906a5b1`)
- **การออกแบบ:** สร้าง visual ประเภท `pivotTable` วางด้านล่างที่พิกัด `Y = 142` ความสูง `145`
- **การแก้ปัญหา `showOnRows: true` ไม่ทำงาน:**
  - เพิ่มฟิลด์ `Dim_Product[New Sub Category Name]` ในช่อง `Rows` เป็น Dummy Category Anchor (พร้อมตั้งชื่อ `displayName: " "` และปิด `rowHeaders`)
  - ทำให้คำสั่ง `"showOnRows": true` ทำงานได้สมบูรณ์ ส่งผลให้ค่าเรียงตัวลงมาในแนวตั้ง 3 บรรทัด (แถว 1: รูปธงชาติ SVG, แถว 2: ราคา Spot, แถว 3: DoD)
- **การแก้ปัญหาธงชาติไม่ขึ้น:**
  - ปรับปรุง Measure `[Product Country Flag SVG]` ให้ประเมินผลจาก `SELECTEDVALUE(Dim_Product[Product Name], "")` โดยตรง ไม่ต้องพึ่งพา `Fact_Rice_Price` ป้องกันปัญหา Context หายเมื่อกรองผ่าน Date Slicer
  - ธงชาติ Vector SVG แสดงผลคมชัดครบทุกประเทศ 100%

---

## 6. บทสรุปสุดท้ายและการแก้ปัญหาขั้นเด็ดขาด (DA-review3 Final Implementation)

จากการทดสอบพฤติกรรมจริงของเอนจิน Power BI Desktop ใน [DA-review3.md](file:///d:/OneDrive%20-%20CPCRT/Job/Power%20BI%20Project/Project-Oryza-Hub/pbip/power-bi-visual/DA-review3.md):

1. **ปัญหา Accent Bar ใน Small Multiples:**
   - ใน `cardVisual` แอตทริบิวต์ `accentBar` ถูกผูกไว้กับ Callout Value แต่ละตัว ทำให้ปรากฏเป็นขีดเทา 2 ขีดคั่นกลางตัวเลข และสีไม่สามารถรับ Filter Context ของ Small Multiples ได้
   - **การแก้ไข:** ปิด `accentBar` (`"show": false`) และปรับดีไซน์เป็นการ์ดคลีนสไตล์ Bloomberg / Executive Command:
     - เส้นขอบการ์ดบางเฉียบ `#CBD5E1` (`1D`)
     - หัวการ์ดชื่อสินค้าสีกรมท่า `#0F172A` (11.5pt)
     - ตัวเลข Spot Price 16pt (Segoe UI Semibold)
     - ตัวเลข DoD % 14pt สี `#475569`
2. **การแก้ปัญหาสัญลักษณ์และการแสดงผล `DoD %` ด้วย Native Format String:**
   - ปรับ Measure `Price Oryza DoD %` ใน `_Measures.tmdl` ให้มีรูปแบบ:
     ```tmdl
     formatString: ▲ 0.00%;▼ 0.00%;● 0.00%
     ```
   - วิธีนี้อาศัยความสามารถของ Analysis Services / DAX Engine โดยตรง ทำให้ตัวเลขมีสัญลักษณ์ **▲ (เมื่อราคาขึ้น)**, **▼ (เมื่อราคาลง)** และ **● 0.00% (เมื่อราคาทรงตัว)** โดยอัตโนมัติ โดยไม่ติดปัญหา Context หรือบั๊กสีของ Small Multiples
3. **การตัด Visual Matrix (`c7a4b8d2e1f34906a5b1`) ออกอย่างสมบูรณ์:**
   - ลบ Visual Matrix ทิ้ง เพื่อคืนพื้นที่และไม่ให้บดบัง Line Chart ด้านล่าง ทำให้ภาพรวมหน้าแดชบอร์ดสะอาด สวยงาม และอ่านง่ายระดับผู้บริหารสูงสุด


