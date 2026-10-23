จากโครงสร้างโมเดล **Oryza Hub** ที่มีอยู่ คุณมีคอลัมน์และตารางมิติที่ทรงพลังซ่อนอยู่อีกหลายตัว (เช่น `Crop Status`, `Dim_Country`, และ `Fact_Exchange_Rate`) ซึ่งสามารถนำมาสร้าง **DAX Measures เชิงลึก** เพื่อใช้วิเคราะห์สถานการณ์ตลาดข้าวในมุมของการค้าส่งออก การตั้งราคา และการจัดซื้อเพิ่มเติมได้อีก 5 มิติหลัก ดังนี้ครับ:

---

## 1. การวิเคราะห์ส่วนต่างราคา "ข้าวเก่า vs ข้าวใหม่" (Crop Status Premium)

ในตาราง `Fact_Rice_Price` มีคอลัมน์ `Crop Status` ("New" / "Old") เตรียมไว้แล้ว ในธุรกิจข้าว ช่วงเปลี่ยนฤดูกาลเก็บเกี่ยวจะมีส่วนต่างราคาของข้าวเก่าและข้าวใหม่ที่ชัดเจน การวัดค่านี้จะช่วยให้ฝ่ายจัดซื้อและฝ่ายขายเห็นจังหวะการระบายสต็อกหรือการเข้าซื้อครับ

- **`[Price Old Crop (THB/kg)]`** : ราคาเฉลี่ยข้าวเก่า

```dax
CALCULATE([Price (THB/kg)], Fact_Rice_Price[Crop Status] = "Old")

```

- **`[Price New Crop (THB/kg)]`** : ราคาเฉลี่ยข้าวใหม่

```dax
CALCULATE([Price (THB/kg)], Fact_Rice_Price[Crop Status] = "New")

```

- **`[Old vs New Crop Spread (THB/kg)]`** : ส่วนต่างราคาข้าวเก่าเทียบข้าวใหม่ (ปกติข้าวเก่าจะมีราคาสูงกว่าในบางสายพันธุ์ เช่น ข้าวหอมมะลิ หรือใช้ดู Discount ของข้าวใหม่ที่เพิ่งออกสู่ตลาด)

```dax
[Price Old Crop (THB/kg)] - [Price New Crop (THB/kg)]

```

---

## 2. ความสามารถในการแข่งขันด้านการส่งออก (Global Export Competitiveness)

เนื่องจากแหล่งข้อมูล **Oryza (`S005`)** มีราคา FOB ของหลายประเทศผู้ส่งออก (ไทย, เวียดนาม, อินเดีย, ปากีสถาน ฯลฯ) เชื่อมกับตาราง `Dim_Country` การสร้างสูตรเปรียบเทียบราคา FOB ของไทยเทียบกับคู่แข่งโดยตรง จะช่วยให้ทีมการตลาดตอบคำถามลูกค้าต่างประเทศได้ทันทีว่าทำไมราคาไทยถึงแพงหรือถูกกว่าคู่แข่งกี่ดอลลาร์ต่อตัน

- **`[FOB Thailand (USD/ton)]`** : ราคาส่งออก FOB ของไทย

```dax
CALCULATE(
    [Price Oryza (USD/ton)],
    Dim_Country[CountryName] = "Thailand"
)

```

- **`[FOB Vietnam (USD/ton)]`** : ราคาส่งออก FOB ของเวียดนาม (สามารถสร้างเพิ่มสำหรับ India / Pakistan ได้ด้วยรูปแบบเดียวกัน)

```dax
CALCULATE(
    [Price Oryza (USD/ton)],
    Dim_Country[CountryName] = "Vietnam"
)

```

- **`[FOB Spread TH vs VN (USD/ton)]`** : ส่วนต่างราคาส่งออกไทยเทียบเวียดนาม
- **ความหมายเชิงธุรกิจ:** ถ้าค่าเป็นบวกสูงๆ แปลว่าข้าวไทยแพงกว่าเวียดนามมาก อาจทำให้เสียเปรียบในการประมูลหรือขายยากขึ้นในกลุ่มข้าวขาว/ข้าวนึ่ง

```dax
[FOB Thailand (USD/ton)] - [FOB Vietnam (USD/ton)]

```

---

## 3. สัญญาณแนวโน้มและตำแหน่งราคา (Market Regime & Price Percentile)

นอกจากมีเส้น Moving Average (7D, 14D, 30D) และ 52W High/Low แล้ว เราสามารถสร้างตัวชี้วัดแบบ **Stochastic / Percentile** เพื่อดูว่าราคาปัจจุบันยืนอยู่ตรงไหนของกรอบราคาในรอบ 1 ปี (0% = ถูกสุดในรอบปี, 100% = แพงสุดในรอบปี) และสร้างสัญญาณบอกแนวโน้มขาขึ้น/ขาลงอัตโนมัติครับ

- **`[52W Price Position %]`** : ดัชนีชี้วัดระดับราคาในกรอบ 52 สัปดาห์ (Stochastic Position)
- **ความหมายเชิงธุรกิจ:** ถ้าน้อยกว่า 20% แปลว่าราคาอยู่ในโซนต่ำของรอบปี (จังหวะน่าสะสมสต็อกของฝ่ายจัดซื้อ) ถ้าเกิน 80% แปลว่าราคาอยู่ในโซนแพง

```dax
VAR _Current = [Price (THB/kg)]
VAR _Low52W =
    CALCULATE(
        MINX(VALUES(Dim_Date[Date]), [Price (THB/kg)]),
        DATESINPERIOD(Dim_Date[Date], MAX(Dim_Date[Date]), -52, WEEK)
    )
VAR _High52W =
    CALCULATE(
        MAXX(VALUES(Dim_Date[Date]), [Price (THB/kg)]),
        DATESINPERIOD(Dim_Date[Date], MAX(Dim_Date[Date]), -52, WEEK)
    )
RETURN
    DIVIDE(_Current - _Low52W, _High52W - _Low52W)

```

- **`[Trend Signal (7D vs 30D)]`** : สัญญาณบอกทิศทางแนวโน้มระยะสั้นเทียบระยะกลาง (สำหรับใส่ในตารางสรุปสินค้า)

```dax
VAR _MA7 = [Price 7D Moving Avg]
VAR _MA30 = [Price 30D Moving Avg]
RETURN
    SWITCH(
        TRUE(),
        ISBLANK(_MA7) || ISBLANK(_MA30), BLANK(),
        _MA7 > _MA30 * 1.01, "▲ ขาขึ้น (Uptrend)",
        _MA7 < _MA30 * 0.99, "▼ ขาลง (Downtrend)",
        "► ทรงตัว (Sideways)"
    )

```

---

## 4. การแยกผลกระทบจากค่าเงินบาท (FX Impact Decomposition)

ในการส่งออกข้าว บ่อยครั้งที่ราคา FOB (USD/ton) ในตลาดโลกถูกกดดันหรือขยับขึ้นเพราะ **"ค่าเงินบาทแข็งหรืออ่อน"** ไม่ใช่เพราะราคาข้าวสารในประเทศเปลี่ยน การวัด % การเปลี่ยนแปลงของค่าเงินควบคู่ไปกับราคาข้าว จะช่วยให้ผู้บริหารแยกแยะสาเหตุได้ชัดเจนครับ

- **`[FX DoD %]`** : อัตราการแข็งค่า/อ่อนค่าของเงินบาทเทียบวันทำการก่อนหน้า

```dax
VAR _CurrentDate = MAX(Dim_Date[Date])
VAR _PrevDate =
    CALCULATE(
        MAX(Fact_Exchange_Rate[Date]),
        Fact_Exchange_Rate[Date] < _CurrentDate,
        Fact_Exchange_Rate[Currency] = "USD",
        REMOVEFILTERS(Dim_Date)
    )
VAR _CurrentFX = [Exchange Rate USD/THB]
VAR _PrevFX =
    CALCULATE(
        AVERAGE(Fact_Exchange_Rate[Sight Bill Buying Rates]),
        Fact_Exchange_Rate[Date] = _PrevDate,
        Fact_Exchange_Rate[Currency] = "USD",
        REMOVEFILTERS(Dim_Date)
    )
RETURN
    DIVIDE(_CurrentFX - _PrevFX, _PrevFX)

```

- **`[FX 30D Change %]`** : การเปลี่ยนแปลงค่าเงินบาทในรอบ 30 วัน (นำมาวางคู่กับ `[Price MoM %]` เพื่อดูว่าที่ราคาส่งออก USD เปลี่ยนแปลง มาจากค่าเงินกี่ %)

```dax
VAR _CurrentFX = [Exchange Rate USD/THB]
VAR _Prev30DFX = CALCULATE([Exchange Rate USD/THB], DATEADD(Dim_Date[Date], -30, DAY))
RETURN
    DIVIDE(_CurrentFX - _Prev30DFX, _Prev30DFX)

```

---

## 5. การวิเคราะห์โครงสร้างราคาข้ามชนิดข้าว (Cross-Product Ratio / Basis)

ในตลาดข้าวไทย มักมีการดูความสัมพันธ์ระหว่างข้าวแต่ละชนิด เช่น **ส่วนต่างระหว่างข้าวหอมมะลิกับข้าวขาว 5%** หรือ **สัดส่วนราคาปลายข้าว (Broken Rice) เทียบกับข้าวต้น (White Rice 5%)** เพื่อดูว่าความต้องการของตลาดกำลังเทไปที่กลุ่มไหน (เช่น ช่วงที่วัตถุดิบอาหารสัตว์แพง ราคาปลายข้าวอาจจะขยับเข้าใกล้ราคาข้าวขาวมากขึ้น)

- **`[Broken to White Rice Ratio %]`** : สัดส่วนราคาปลายข้าวเทียบข้าวขาว (ปรับเปลี่ยนรหัส `Product Code` ตามรหัสจริงในตาราง `Dim_Product` ของคุณได้เลยครับ)

```dax
VAR _BrokenRicePrice =
    CALCULATE([Price DIT (THB/kg)], Dim_Product[Product Code] = "รหัสปลายข้าว A1 เลิศ")
VAR _WhiteRice5Price =
    CALCULATE([Price DIT (THB/kg)], Dim_Product[Product Code] = "รหัสข้าวขาว 5%")
RETURN
    DIVIDE(_BrokenRicePrice, _WhiteRice5Price)

```

---

### 💡 สรุปการจัดหมวดหมู่เข้า Display Folders เดิมของคุณ:

- กลุ่ม **Crop Status**, **Global Competitiveness**, และ **Cross-Product Ratio** $\rightarrow$ สามารถเพิ่มเข้าใน **`📂 04 Marketing & Price Spreads`**

- กลุ่ม **52W Position %** และ **Trend Signal** $\rightarrow$ สามารถเพิ่มเข้าใน **`📂 05 Moving Averages & Trend`**

- กลุ่ม **FX Change %** $\rightarrow$ สามารถเพิ่มเข้าใน **`📂 02 Currency & Unit Conversion`** หรือ **`📂 06 Time Intelligence`**

คุณสนใจอยากเจาะลึกกลุ่มไหนเป็นพิเศษ หรือต้องการนำรหัสสินค้าจริงใน `Dim_Product` มาผูกสูตรเปรียบเทียบคู่สายพันธุ์หลักของบริษัทไหมครับ?
