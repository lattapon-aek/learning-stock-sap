# SAP Stock & Material Master สำหรับ Developer
### Dohome Development Team

---

## PART 0 — Hook

---

### Slide 1 — เปิดด้วยคำถาม

**[Visual: ภาพ 2 ตัวเลขขนาดใหญ่ เทียบกัน]**

> **"ทำไม stock ที่ integration ดึงมา**
> **กับ stock ที่ขายได้จริง ถึงไม่ตรงกัน?"**

ตัวอย่าง:
- ระบบ integration เห็น stock = **100 ชิ้น**
- ลูกค้าสั่งซื้อได้จริงแค่ **67 ชิ้น**

**คำตอบไม่ใช่ "data ผิด" — แต่เป็นเพราะเราดูผิดตัวเลข**

> *(session นี้จะตอบคำถามนี้ และอีกหลายคำถามที่ทีมเจออยู่จริง)*

---

## PART 1 — SAP มอง "Stock" ยังไง

---

### Slide 2 — Stock ใน SAP ไม่ใช่แค่จำนวน

**[Visual: สมการใหญ่กลาง slide]**

```
มูลค่าสินค้าคงคลัง = จำนวน × ราคาประเมิน
Stock Value        = Quantity × Valuation Price
```

**สิ่งที่ต้องเข้าใจก่อนทุกอย่าง:**
- SAP เก็บสินค้าทุกชิ้นในรูปแบบ **จำนวน + มูลค่า** พร้อมกันเสมอ
- เมื่อสินค้าเคลื่อนไหว ระบบอัปเดตทั้งสองอย่างพร้อมกัน
- ทีม Dev ที่บอกว่า "stock ผิด" ต้องถามก่อนว่า — **ผิดที่จำนวน หรือมูลค่า หรือทั้งคู่?**

> เพราะ root cause และจุด debug ไม่เหมือนกันเลย

---

### Slide 3 — Stock มีหลายประเภท — ไม่ใช่ตัวเลขเดียว

**[Visual: แผนภูมิวงกลมซ้อนกัน หรือ stacked bar แบ่ง stock types]**

```
Stock ทั้งหมดที่มีใน SAP (Total Stock)
├── Unrestricted Stock       → พร้อมขาย พร้อมใช้ได้ทันที
│   └── Reserved (จอง)       → ถูกจองแล้ว (เช่น Delivery) ยังนับเป็น Unrestricted
├── Quality Inspection       → รอตรวจรับ ยังใช้ไม่ได้
└── Blocked Stock            → ระงับการใช้งาน
```

**ตัวอย่าง Dohome — ทำไมตัวเลขไม่ตรง:**

| ประเภท | จำนวน |
|---|---:|
| Unrestricted Stock | 100 |
| ถูกจองโดย Delivery | −33 |
| **Available to Promise** | **67** |

> Integration ที่ดึงแค่ `Unrestricted` จะเห็น 100 แต่ขายได้จริงแค่ 67

---

### Slide 4 — 4 คำถามก่อน Debug Stock ทุกครั้ง

**[Visual: 4 กล่องคำถาม เรียงเป็น grid 2×2]**

| คำถาม | ทำไมต้องถาม |
|---|---|
| **1. Material Type คืออะไร?** | บางประเภทไม่มีมูลค่า หรือไม่เข้า stock เลย |
| **2. Price Control เป็น V หรือ S?** | กำหนดว่าราคาเปลี่ยนยังไงเมื่อรับสินค้า |
| **3. Movement นี้เข้าหรือออก stock?** | บาง movement เข้า consumption ไม่ใช่ stock |
| **4. อยู่ใน Valuation Area เดียวกันหรือเปล่า?** | กำหนดว่าการโอนสินค้ากระทบบัญชีหรือไม่ |

> ถ้าตอบ 4 ข้อนี้ได้ — Debug เรื่อง stock costing จะไม่ใช่การเดาสุ่มอีกต่อไป

---

## PART 2 — Article / Material Master

---

### Slide 5 — Material Master คือ "ศูนย์กลางข้อมูลสินค้า"

**[Visual: diagram วงกลม Material Master อยู่กลาง มี module รอบ]**

```
                    Purchasing
                        ↑
           MRP ←  [Material Master]  → Sales
                        ↓
               Inventory / Accounting
```

**หลักการสำคัญ:**
- ข้อมูลสินค้าทุกอย่างอยู่ที่นี่ที่เดียว — ทุก module ดึงจากที่เดียวกัน
- SAP Retail เรียกว่า **Article Master** แต่หลักการ valuation เหมือนกันทุกอย่าง
- ไม่ใช่ 1 table — แต่แยกเก็บเป็น **หลาย segments ตามระดับองค์กร**

> ถ้าข้อมูลผิดที่ Material Master — ทุก module ได้รับข้อมูลผิดพร้อมกัน

---

### Slide 6 — โครงสร้างตาราง Material Master

**[Visual: ER Diagram แบบง่าย แสดง hierarchy จาก MARA ลงมา]**

```
MARA  (ระดับ Client — ข้อมูลทั่วไปของสินค้า)
 └── MARC  (ระดับ Plant — ข้อมูลของ DC/สาขา)
      ├── MARD  (ระดับ Storage Location — ปริมาณในคลัง)
      └── MBEW  (ระดับ Valuation Area — ราคาและมูลค่า)
```

**ระดับองค์กรของ Dohome:**

| ระดับ | ตัวอย่าง |
|---|---|
| Client | ระบบ SAP ทั้งระบบ |
| Company Code | บริษัท Dohome |
| Plant | DC บางนา, DC ลาดกระบัง, สาขาต่างๆ |
| Storage Location | คลัง A, คลัง B, โซนรับสินค้า |

> **Gotcha**: อย่า Query แค่ `MATNR` — ข้อมูลส่วนใหญ่ต้องใส่ Plant หรือ Valuation Area ด้วย

---

### Slide 7 — Fields ใน MBEW ที่ Integration ต้องรู้

**[Visual: ตาราง highlight fields สำคัญ]**

`MBEW` = ตารางที่เก็บ **ราคาและมูลค่าสินค้า** ระดับ Valuation Area

| Field | ความหมาย | ค่าที่เป็นไปได้ |
|---|---|---|
| `MBEW-BKLAS` | Valuation Class — ใช้เลือก G/L Account | เช่น 3000, 3100 |
| `MBEW-VPRSV` | Price Control | `V` = Moving Average, `S` = Standard |
| `MBEW-VERPR` | Moving Average Price ปัจจุบัน | ตัวเลขราคา |
| `MBEW-STPRS` | Standard Price | ตัวเลขราคา |
| `MBEW-LBKUM` | Total Valuated Stock | จำนวนรวมที่ตีมูลค่า |
| `MBEW-SALK3` | Total Stock Value | มูลค่ารวม |

**การอ่านราคาที่ถูกต้อง:**
```
ถ้า VPRSV = 'V'  → ใช้ VERPR (Moving Average Price)
ถ้า VPRSV = 'S'  → ใช้ STPRS (Standard Price)
```

---

### Slide 8 — Material Type มีผลต่อพฤติกรรม Stock

**[Visual: ตาราง 4 แถว highlight แถวที่ dev มักพลาด]**

| Material Type | ความหมาย | มีปริมาณ? | มีมูลค่า? |
|---|---|:---:|:---:|
| HAWA (Trading Goods) | สินค้าซื้อมาขายต่อ เช่น สินค้า Dohome | ✅ | ✅ |
| ROH (Raw Material) | วัตถุดิบ | ✅ | ✅ |
| NLAG (Non-Stock) | วัสดุสิ้นเปลืองที่ไม่เก็บเป็น stock | ❌ | ❌ |
| UNBW (Unvaluated) | ติดตามจำนวนแต่ไม่ตีมูลค่า | ✅ | ❌ |

> **สำคัญ**: `NLAG` เมื่อทำ GR จะเข้า **consumption โดยตรง** — ไม่มีใน stock เลย
> Integration ที่คาดหวัง "สินค้าทุกตัวเข้า stock" จะผิดทันทีเมื่อเจอ NLAG

---

### Slide 9 — Valuation Area คืออะไร และทำไมสำคัญกับ Dohome

**[Visual: แผนภาพ plant map ของ Dohome แสดง valuation area]**

**Valuation Area** = ระดับที่ SAP ใช้ตีมูลค่าสินค้า

มีสองแบบ:
- **ระดับ Company Code** — Plant ทั้งหมดใน Dohome ใช้ราคาเดียวกัน
- **ระดับ Plant** — แต่ละ DC/สาขามีราคาของตัวเอง

**ผลกระทบต่อ Dohome:**

| สถานการณ์ | Valuation Area = Company Code | Valuation Area = Plant |
|---|---|---|
| โอนสินค้า DC → สาขา (ใน company เดียว) | ไม่มี FI Document | มี FI Document |
| ราคาสินค้าระหว่าง DC กับสาขา | เหมือนกัน | อาจต่างกัน |

> Dohome ต้องรู้ว่าใช้แบบไหน — เพราะมีผลต่อการออกแบบ integration ทั้งหมด

---

## PART 3 — Price Control และ Moving Average Price

---

### Slide 10 — ทำไม Price Control ถึงสำคัญที่สุด?

**[Visual: ลูกศรจาก Price Control ไปหาผลกระทบ 4 อย่าง]**

**Price Control** กำหนดสิ่งเหล่านี้ทั้งหมด:

```
Price Control (V หรือ S)
    ├── มูลค่าสินค้าในคลังคำนวณยังไง
    ├── Invoice ที่ราคาต่างจาก PO → ไปไหน?
    ├── บัญชีไหนถูก Debit/Credit เมื่อสินค้าเคลื่อนไหว
    └── ตัวเลขใน MBEW-VERPR เปลี่ยนเมื่อไหร่
```

> ถ้าไม่รู้ Price Control ของสินค้า — อธิบายพฤติกรรมของ stock ไม่ได้เลย

---

### Slide 11 — Moving Average Price (V) — หลักการ

**[Visual: สูตรใหญ่กลาง slide + ไดอะแกรมแสดงการไหลของมูลค่า]**

**แนวคิดหลัก**: ราคาสินค้าสะท้อนต้นทุนจริงที่รับเข้ามาล่าสุด

**สูตร:**
```
MAP ใหม่ = (มูลค่า stock เดิม + มูลค่าที่รับเข้าใหม่)
           ────────────────────────────────────────────
                    จำนวน stock รวมทั้งหมด
```

**MAP เปลี่ยนเมื่อ:**
- ✅ รับสินค้าเข้าจาก PO (Goods Receipt — Movement 101)
- ✅ บันทึก Invoice ที่ราคาต่างจาก PO (ถ้า stock เพียงพอ)
- ❌ ขายออก (Movement 601) — ตัดที่ MAP ปัจจุบัน MAP ไม่เปลี่ยน
- ❌ โอนระหว่างคลัง (Movement 311) — ไม่กระทบมูลค่า

---

### Slide 12 — ตัวอย่าง Dohome: รับสินค้าเข้า DC และ MAP เปลี่ยนยังไง

**[Visual: ตารางแสดง step-by-step พร้อม highlight ตัวเลขที่เปลี่ยน]**

*สถานการณ์: DC รับสินค้าประเภท HAWA (Price Control = V)*

| ขั้นตอน | การเปลี่ยนแปลง | จำนวน | มูลค่ารวม | MAP |
|---|---|---:|---:|---:|
| เริ่มต้น | — | 100 | 10,000 | **100.00** |
| GR: รับเข้า 50 ชิ้น @120 บาท | +50 ชิ้น, +6,000 บาท | 150 | 16,000 | **106.67** |
| Invoice: ราคาจริง @115 บาท | ปรับมูลค่า −250 บาท | 150 | 15,750 | **105.00** |
| GI: ขายออก 80 ชิ้น | −80 ชิ้น @105 บาท | 70 | 7,350 | **105.00** |
| GR: รับเข้าอีก 30 ชิ้น @130 บาท | +30 ชิ้น, +3,900 บาท | 100 | 11,250 | **112.50** |

**สิ่งที่ควรสังเกต:**
- MAP เปลี่ยนเฉพาะตอนรับเข้าและ Invoice — ไม่เปลี่ยนตอนขายออก
- ยอดขายตัดต้นทุนด้วย MAP ณ เวลานั้น

---

### Slide 13 — ตัวอย่าง Dohome: ขายสินค้า → ผลต่อ Stock และบัญชี

**[Visual: Flowchart แสดงการตัดต้นทุนเมื่อขาย]**

*สถานการณ์: สาขา Dohome ขายสินค้า MAP = 105 บาท จำนวน 10 ชิ้น*

**ขั้นตอนใน SAP:**
1. ระบบขาย (SD) สร้าง Delivery
2. Post Goods Issue (Movement **601**)
3. SAP ตัดต้นทุนอัตโนมัติ

**ผลทางบัญชี:**
```
Dr  ต้นทุนขาย (COGS)          1,050  บาท   ← 10 × MAP 105
    Cr  สินค้าคงคลัง (Stock)  1,050  บาท
```

**สิ่งที่ต้องรู้สำหรับ Integration:**
- MAP ที่ใช้ตัดคือ MAP **ณ เวลาที่ post** — ไม่ใช่ราคาขาย
- ถ้า integration ส่ง Movement 601 ช้ากว่าความเป็นจริง → MAP อาจไม่ตรงกับที่คาด

---

### Slide 14 — Standard Price (S) — เมื่อราคาไม่เปลี่ยนตาม market

**[Visual: เปรียบเทียบ 2 กรณีแบบ side-by-side]**

**แนวคิดหลัก**: ราคาสินค้าคงที่ตามที่กำหนดไว้ — ส่วนต่างไปบัญชี PRD แยก

*ตัวอย่าง: Standard Price = 100 บาท, รับสินค้าเข้า 50 ชิ้น ราคา PO = 120 บาท*

```
Dr  สินค้าคงคลัง (BSX)          5,000  บาท   ← 50 × Standard 100
Dr  ส่วนต่างราคา PRD            1,000  บาท   ← ส่วนเกินจาก PO
    Cr  GR/IR Clearing (WRX)    6,000  บาท   ← 50 × PO 120
```

**เปรียบเทียบ V vs S:**

| ประเด็น | Moving Average (V) | Standard Price (S) |
|---|---|---|
| ราคาสินค้าในคลัง | เปลี่ยนตามของจริง | คงที่ตลอด |
| เมื่อ Invoice ราคาต่างจาก PO | ปรับ stock value | ไปบัญชี PRD |
| เหมาะกับ | ค้าปลีก สินค้าราคาผันผวน | สินค้าผลิต ต้องการต้นทุนคงที่ |
| Dohome ใช้กับ | สินค้าทั่วไป (HAWA) | — |

---

### Slide 15 — สรุป: Movement ไหนกระทบ MAP บ้าง?

**[Visual: ตารางใหญ่ชัดเจน พร้อม ✅ ❌]**

| Movement | ความหมาย | กระทบ MAP? | เหตุผล |
|---|---|:---:|---|
| **101** | รับสินค้าเข้าจาก PO | ✅ | มูลค่าใหม่เพิ่มเข้า stock |
| **102** | ยกเลิกการรับเข้า | ✅ | ย้อนกลับการเปลี่ยน MAP |
| **122** | คืนสินค้ากลับ vendor | ✅ | มูลค่าออกจาก stock |
| **201** | เบิกใช้ภายใน (Cost Center) | ❌ | ตัดที่ MAP ปัจจุบัน |
| **311** | โอนระหว่างคลังใน Plant เดียว | ❌ | ไม่มี FI document |
| **351** | DC → สาขา (STO) | ❌ (ที่ issuing plant) | มูลค่าออกไป in-transit |
| **551** | ตัดทิ้ง (Scrap) | ❌ | ตัดที่ MAP ปัจจุบัน |
| **601** | ขายออก (Delivery) | ❌ | ตัดที่ MAP ปัจจุบัน |
| **Invoice (MIRO)** | บันทึกใบแจ้งหนี้ | ✅ (ถ้า V + stock พอ) | ปรับส่วนต่างราคาเข้า stock |

> **จำง่ายๆ**: MAP เปลี่ยนเฉพาะตอน "ของเข้า + มูลค่าเข้า" หรือ "Invoice ปรับราคา"

---

## PART 4 — Procure-to-Pay Flow

---

### Slide 16 — ภาพรวม Procure-to-Pay Flow

**[Visual: Flowchart แนวนอน พร้อมบอกว่า step ไหนสร้าง document อะไร]**

```
[PR]──→[PO]──→[Goods Receipt]──→[Invoice Receipt]──→[Payment]
  ไม่มี FI    ไม่มี FI    มี Material Doc       มี FI Doc       มี FI Doc
                          + FI Doc (ถ้า valuated)
```

**เอกสารที่เกิดในแต่ละขั้น:**

| ขั้นตอน | Transaction | เอกสารที่สร้าง | ตาราง |
|---|---|---|---|
| Purchase Requisition | ME51N | PR Document | EBAN |
| Purchase Order | ME21N | PO Document | EKKO / EKPO |
| Goods Receipt | MIGO | Material Doc + FI Doc | MKPF/MSEG + BKPF/BSEG |
| Invoice Receipt | MIRO | FI Doc (Invoice) | RBKP/RSEG + BKPF/BSEG |
| Payment | F110 | FI Doc (Payment) | BKPF/BSEG |

> **PR และ PO ไม่สร้าง FI Document** — มีผลแค่ฝั่ง Logistics

---

### Slide 17 — GR/IR Clearing Account คืออะไร?

**[Visual: แผนภาพแสดงการไหลของ debit/credit ระหว่าง GR และ IR]**

**GR/IR Clearing** = บัญชีพักชั่วคราว ระหว่างรับของกับรับใบแจ้งหนี้

```
ตอน Goods Receipt (101):
    Dr  สินค้าคงคลัง       10,000
        Cr  GR/IR Clearing  10,000  ← ค้างไว้รอ Invoice

ตอน Invoice Receipt (MIRO):
    Dr  GR/IR Clearing     10,000  ← เคลียร์ออก
        Cr  เจ้าหนี้ (AP)  10,000
```

**ผลลัพธ์**: GR/IR = 0 เมื่อมี GR และ Invoice ครบ

**ตัวอย่าง Dohome — เมื่อ GR/IR ไม่เป็นศูนย์:**

| กรณี | สาเหตุ |
|---|---|
| GR/IR มียอด Debit ค้าง | รับของเข้าแล้ว แต่ยังไม่ได้รับ Invoice |
| GR/IR มียอด Credit ค้าง | Invoice มาแล้ว แต่ของยังไม่มาถึง |

---

### Slide 18 — ตัวอย่างจริง: รับสินค้าเข้า DC จาก Vendor

**[Visual: Timeline แนวนอน แสดง 3 ขั้นตอน]**

*สถานการณ์: DC สั่งซื้อสินค้า 100 ชิ้น ราคา PO = 100 บาท/ชิ้น, MAP ปัจจุบัน = 95 บาท*

**ขั้นที่ 1 — สร้าง PO (ME21N):**
- SAP บันทึกว่าจะสั่งซื้อ 100 ชิ้น @100 บาท
- ไม่มีผลต่อ stock และบัญชีใดๆ

**ขั้นที่ 2 — รับสินค้า GR (MIGO — Movement 101):**
```
Dr  สินค้าคงคลัง (BSX)   10,000  บาท
    Cr  GR/IR (WRX)       10,000  บาท
```
- MAP ใหม่คำนวณอัตโนมัติ: (9,500 + 10,000) ÷ (100 + 100) = **97.50 บาท**

**ขั้นที่ 3 — รับ Invoice (MIRO):**
- ถ้า Invoice = PO price (100 บาท):
```
Dr  GR/IR (WRX)     10,000  บาท
    Cr  เจ้าหนี้    10,000  บาท
```
- GR/IR เคลียร์ = 0 ✅

---

### Slide 19 — เมื่อ Invoice ราคาต่างจาก PO

**[Visual: แผนภาพ 2 กรณี V vs S แบบ side-by-side]**

*สถานการณ์: PO = 100 บาท แต่ Invoice จริง = 110 บาท (100 ชิ้น)*

**กรณีที่ 1 — Price Control = V (Moving Average):**
```
Dr  GR/IR (WRX)              10,000
Dr  สินค้าคงคลัง (BSX)        1,000  ← ปรับมูลค่า stock เพิ่ม
    Cr  เจ้าหนี้              11,000
```
- MAP เปลี่ยน (ถ้า stock ยังมีเพียงพอ)

**กรณีที่ 2 — Price Control = S (Standard):**
```
Dr  GR/IR (WRX)              10,000
Dr  ส่วนต่างราคา (PRD)        1,000  ← ไม่กระทบ stock value
    Cr  เจ้าหนี้              11,000
```
- Stock value ไม่เปลี่ยน

> **จุดที่ integration ต้องระวัง**: ถ้าส่ง Invoice สาย MAP อาจเปลี่ยนย้อนหลัง

---

### Slide 20 — Material Doc ≠ FI Doc — อย่า Join ด้วยเลขเดียวกัน

**[Visual: ภาพ 2 กล่อง แสดง document numbers ต่างกัน พร้อม reference key ที่ใช้ join]**

**สิ่งที่เกิดขึ้นเมื่อ Post GR:**

| เอกสาร | เลขเอกสาร | ตาราง | ความหมาย |
|---|---|---|---|
| Material Document | 5000012345 | MKPF / MSEG | บันทึกการเคลื่อนไหวสินค้า |
| Accounting Document | 4900056789 | BKPF / BSEG | บันทึกรายการบัญชี |

**เลขทั้งสองไม่เท่ากัน!**

วิธีเชื่อมที่ถูกต้อง:
- ใช้ `MSEG-LFBNR` หรือ reference fields ที่ SAP กำหนดให้
- ไม่ใช่การ join โดยตรงจากเลขเอกสาร

---

## PART 5 — Movement Types และ Stock ของ Dohome

---

### Slide 21 — Movement Type คืออะไร?

**[Visual: กล่องใหญ่ Movement Type 101 อยู่กลาง แสดงว่าควบคุมอะไรบ้าง]**

**Movement Type** = รหัส 3 หลักที่บอก SAP ว่า "เกิดอะไรกับสินค้า"

```
Movement Type
    ├── อัปเดต Quantity ไหม?
    ├── อัปเดต Value ไหม?
    ├── สร้าง FI Document ไหม?
    ├── ใช้บัญชี G/L ไหน?
    └── อ้างอิง Document อะไร (PO / Order / Delivery)?
```

**Pattern ของเลข:**
- **1xx** = รับเข้า (Goods Receipt)
- **2xx** = เบิกออก (Goods Issue to consumption)
- **3xx** = โอนย้าย (Transfer Posting)
- **5xx** = รับเข้า/จ่ายออกพิเศษ
- **6xx** = จ่ายออกเพื่อส่งลูกค้า (Delivery)

---

### Slide 22 — Movement Types ที่ Dohome ต้องรู้

**[Visual: ตารางใหญ่ highlight แถวที่เกี่ยวกับ Dohome โดยตรง]**

| Movement | ความหมาย | กระทบ Stock Value | กระทบ MAP | มี FI Doc? |
|---|---|:---:|:---:|:---:|
| **101** | รับสินค้าจาก PO (DC รับของจาก vendor) | ✅ | ✅ | ✅ |
| **102** | ยกเลิก GR | ✅ | ✅ | ✅ |
| **122** | คืนสินค้าให้ vendor | ✅ | ✅ | ✅ |
| **201** | เบิกใช้ภายใน (Cost Center) | ✅ | ❌ | ✅ |
| **311** | โอนระหว่างคลังย่อยใน Plant เดียวกัน | ❌ | ❌ | ❌ |
| **351** | โอน DC → สาขา (Stock Transport Order) | ✅ | ❌ | ✅ |
| **551** | ตัดทิ้ง / สินค้าเสียหาย (Scrap) | ✅ | ❌ | ✅ |
| **601** | ขายออก (Goods Issue for Delivery) | ✅ | ❌ | ✅ |

---

### Slide 23 — Dohome Scenario: โอนสินค้า DC → สาขา (STO Flow)

**[Visual: Flowchart แสดงการเดินทางของสินค้าและมูลค่า]**

```
[DC สร้าง STO] → [DC ส่งของออก Movement 351] → [สาขารับของเข้า]
                         ↓
               Stock ออกจาก DC → ไป "Stock in Transit"
               FI Document สร้างที่ฝั่ง DC ทันที
```

**ผลที่เกิดขึ้น:**

| ขั้นตอน | Stock DC | Stock in Transit | Stock สาขา |
|---|---:|---:|---:|
| ก่อนโอน | 100 | 0 | 50 |
| หลัง Movement 351 (DC ส่งออก) | 70 | 30 | 50 |
| หลังสาขารับเข้า | 70 | 0 | 80 |

> **สิ่งสำคัญ**: มูลค่าเดินทางตั้งแต่ **DC ส่งออก** ไม่ใช่ตอนสาขารับเข้า
> Integration ที่คิดว่า "ของถึงร้านเมื่อไร ค่อยมีมูลค่า" จะคำนวณผิด

---

### Slide 24 — Dohome Scenario: DC-DC Transfer

**[Visual: แผนภาพ 2 DC โอนของระหว่างกัน]**

*สถานการณ์: DC บางนา โอนสินค้าให้ DC ลาดกระบัง*

**สิ่งที่ต้องรู้ก่อน:**
- ทั้ง 2 DC อยู่ใน **Valuation Area เดียวกัน** หรือเปล่า?

| Valuation Area | ผลของการโอน |
|---|---|
| เดียวกัน (Company Code level) | ไม่มี FI Document — แค่เปลี่ยน plant |
| ต่างกัน (Plant level) | มี FI Document — มูลค่าออกจาก DC ต้นทาง |

**กรณี Dohome (ต้องตรวจสอบ configuration จริง):**
- ถ้าโอนข้าม Valuation Area → ต้องใช้ STO (Stock Transport Order)
- ถ้าโอนใน Valuation Area เดียว → สามารถใช้ Transfer Posting ได้

---

### Slide 25 — Stock จอง (Reserved Stock) ใน Dohome

**[Visual: แผนภาพแสดง stock pool แบ่งส่วน unrestricted และ reserved]**

**ใน Dohome: เมื่อสร้าง Delivery — SAP จอง Stock อัตโนมัติ**

```
Unrestricted Stock ทั้งหมด = 100 ชิ้น
├── Reserved โดย Delivery #001 = 30 ชิ้น
├── Reserved โดย Delivery #002 = 20 ชิ้น
└── Available to Promise (ATP)  = 50 ชิ้น ← ขายได้จริง
```

**ลำดับการเปลี่ยนแปลง:**

| เหตุการณ์ | Unrestricted | Reserved | ATP |
|---|---:|---:|---:|
| เริ่มต้น | 100 | 0 | 100 |
| สร้าง Delivery #001 (30 ชิ้น) | 100 | 30 | 70 |
| Post Goods Issue (Movement 601) | 70 | 0 | 70 |

**สำคัญสำหรับ Integration:**
- ดึง `MARD-LABST` = Unrestricted (ยังรวม reserved อยู่)
- ต้องดู **Available Quantity** จาก ATP logic — ไม่ใช่แค่ MARD
- ตาราง stock reservation: `RESB`, `VBFA`

---

### Slide 26 — Gotcha ที่ทีม Dev มักพลาด

**[Visual: Warning icon + ตาราง gotcha]**

**⚠️ 1 — Movement 101 ไม่ได้แปลว่าเข้า Stock เสมอไป**

```
PO ปกติ          → Movement 101 → เข้า Unrestricted Stock ✅
PO + Account Assignment (เช่น Cost Center) → Movement 101 → เข้า Consumption ทันที ❌
```

**⚠️ 2 — Movement 102 ≠ Movement 122**

| | 102 | 122 |
|---|---|---|
| ความหมาย | ยกเลิก GR (Cancellation) | คืนสินค้าให้ vendor (Return Delivery) |
| อ้างอิงราคา | จาก Original GR | จาก PO History |
| ใช้เมื่อ | โพสต์ผิด ต้องแก้ | ของเสีย/ไม่ตรงสเปค ต้องคืน |

**⚠️ 3 — ลืม BAPI_TRANSACTION_COMMIT**

```
BAPI_GOODSMVT_CREATE   ← สร้างเอกสาร
BAPI_TRANSACTION_COMMIT ← ต้องเรียกเสมอ!
```
> ถ้าไม่ commit: interface บอก success แต่หา document ไม่เจอ

---

## PART 6 — Account Determination

---

### Slide 27 — SAP ตัดบัญชีอัตโนมัติยังไง?

**[Visual: ลูกศร chain จาก Movement Type ไปยัง G/L Account]**

**ห่วงโซ่ Account Determination:**

```
Movement Type
     ↓
Value String  (กำหนดใน config)
     ↓
Transaction/Event Key
(BSX / WRX / PRD / GBB / KDM / UMB)
     ↓
Valuation Class  (จาก MBEW-BKLAS)
     ↓
G/L Account  (กำหนดใน OBYC)
```

> Dev ไม่ต้องจำทุก node — แต่ต้องรู้ว่า **Valuation Class** และ **Movement Type** คือตัวแปรหลักที่กำหนดบัญชี

---

### Slide 28 — Transaction Keys ที่ต้องรู้จัก

**[Visual: ตาราง 6 keys พร้อม icon สีต่างกัน]**

| Key | ความหมาย | ใช้เมื่อ |
|---|---|---|
| **BSX** | Inventory Posting | ทุกครั้งที่ stock เพิ่ม/ลดมูลค่า |
| **WRX** | GR/IR Clearing | ทุก GR จาก PO และ Invoice |
| **PRD** | Price Difference | ราคาซื้อ ≠ ราคามาตรฐาน (S) หรือ stock ไม่พอรองรับ variance (V) |
| **GBB** | Offsetting Entry | เบิกใช้ภายใน, scrap, delivery to customer |
| **KDM** | Exchange Rate Difference | กรณีซื้อสินค้าสกุลเงินต่างประเทศ |
| **UMB** | Revaluation | ปรับมูลค่าย้อนหลัง, backdated posting |

---

### Slide 29 — ตัวอย่างครบวงจร: GR และ IR ลงบัญชียังไง

**[Visual: 2 กล่องคู่กัน แสดง Dr/Cr ของ GR และ IR]**

*สินค้า Price Control = V, PO price = 100 บาท, MAP ปัจจุบัน = 95 บาท*

**Goods Receipt (Movement 101):**
```
Dr  สินค้าคงคลัง  BSX     10,000  บาท   ← 100 ชิ้น × PO price 100
    Cr  GR/IR    WRX     10,000  บาท
```
→ MAP ใหม่คำนวณอัตโนมัติ

**Invoice Receipt (MIRO) ราคา Invoice = 105 บาท:**
```
Dr  GR/IR       WRX     10,000  บาท
Dr  สินค้าคงคลัง BSX        500  บาท   ← ส่วนต่าง 5×100 ปรับ stock (V)
    Cr  เจ้าหนี้           10,500  บาท
```
→ MAP ปรับอีกครั้งจาก Invoice

---

## PART 7 — Developer Quick Reference

---

### Slide 30 — ถ้าสงสัยเรื่อง... เช็คที่ไหน?

**[Visual: ตาราง 2 คอลัมน์ ใช้เป็น cheat sheet]**

| ถ้าสงสัยเรื่อง… | เช็คก่อนที่ | หมายเหตุ |
|---|---|---|
| Stock value แปลก | `MBEW-VPRSV`, `MBEW-VERPR` | ดู price control + MAP ปัจจุบัน |
| On-hand ไม่ตรง | `MARD-LABST` + stock type + special stock | อย่าดูแค่ LABST |
| Available ≠ Unrestricted | Reservation / ATP | ดู RESB, VBFA |
| GR เข้า stock หรือ consumption | PO account assignment + material type | ดู EKPO-KNTTP |
| Invoice variance ไปไหน | Price control + stock coverage ตอน IR | V vs S ต่างกัน |
| DC → สาขา มี FI ไหม | Valuation Area ของสอง plant | ดู config |
| Material doc กับ FI doc | อย่า join ด้วยเลขเดียวกัน | ใช้ reference key |
| MAP เปลี่ยนแต่ downstream ไม่รู้ | Change pointer ไม่เสมอ | อ่านจาก MATDOC/docs แทน |
| Interface บอก success แต่ไม่มี doc | ลืม BAPI_TRANSACTION_COMMIT | ต้องเรียกทุกครั้ง |
| S/4HANA อ่าน movement docs | ใช้ `MATDOC` | ไม่ใช่ MKPF/MSEG |

---

### Slide 31 — BAPI Shortlist สำหรับ Dohome Integration

**[Visual: ตาราง BAPI พร้อมเส้น flow แสดงว่าใช้ตอนไหน]**

| BAPI | ใช้ทำอะไร | ใช้ใน Flow |
|---|---|---|
| `BAPI_MATERIAL_SAVEDATA` | สร้าง/แก้ Material Master | Article/Stock Master sync |
| `BAPI_MATERIAL_GET_ALL` | อ่าน Material Master ครบชุด | Master data integration |
| `BAPI_GOODSMVT_CREATE` | Post Goods Movement | GR, GI, Transfer |
| `BAPI_GOODSMVT_CANCEL` | ยกเลิก Goods Movement | Reversal |
| `BAPI_PO_CREATE1` | สร้าง Purchase Order | Purchasing integration |
| `BAPI_PO_CHANGE` | แก้ไข Purchase Order | PO amendment |
| `BAPI_INCOMINGINVOICE_CREATE` | บันทึก Invoice Receipt | Invoice integration |
| **`BAPI_TRANSACTION_COMMIT`** | **Commit LUW** | **ต้องเรียกหลัง BAPI ทุกตัว!** |

**ลำดับการเรียก BAPI ที่ถูกต้อง:**
```
1. BAPI_GOODSMVT_CREATE    → ได้ Material Doc Number กลับมา
2. ตรวจ Return Table       → ถ้ามี Error ห้าม Commit
3. BAPI_TRANSACTION_COMMIT → Commit เมื่อไม่มี Error เท่านั้น
```

---

## Slide 32 — สรุป Key Takeaways

**[Visual: 5 กล่อง สรุปจุดสำคัญ]**

**5 สิ่งที่ต้องจำกลับไป:**

1. **Stock = Quantity + Value** เสมอ — ไม่ใช่แค่จำนวน
2. **Price Control (V/S)** กำหนดทุกอย่าง — MAP เปลี่ยนตาม actual, Standard แยก variance
3. **Movement Type** บอกว่า stock เข้าหรือออก และกระทบบัญชีไหน
4. **Stock ที่เห็น ≠ Stock ที่ขายได้** — ต้องดู ATP ไม่ใช่แค่ Unrestricted
5. **BAPI ต้อง COMMIT** — ไม่เช่นนั้น document ไม่เกิด

---

*จบ Slide Deck — SAP Stock & Material Master for Dohome Developers*
*เวลารวม: ~55 นาที | จำนวน Slide: 32*
