# Subscription Flow

## 1) สรุปฟีเจอร์

- ลูกค้าสมัครแพ็กเกจแล้ว ระบบจะสร้าง "บิล" ทันที
- เมื่อจ่ายสำเร็จ สถานะ subscription จะเปิดใช้งาน และออกใบเสร็จ
- ระบบจะสร้างบิลรอบถัดไปอัตโนมัติ (upcoming)
- ต้นเดือน ระบบจะย้ายบิลรอบถัดไปมาเป็นบิลที่ต้องจ่าย (issued)
- ถ้าจ่ายไม่สำเร็จ จะมีช่วงค้างชำระ (`subscription.status = SCHEDULED`, `subscription.paymentStatus = PENDING`) และสุดท้ายถูกพักใช้งาน (`subscription.status = SUSPENDED`)
- ถ้าอัปเกรดกลางเดือน จะมีการคำนวณส่วนลดแบบ prorate + เครดิตจากบิลเก่า
- ถ้าเกิด usage เกินโควตา (overage) ระบบจะเติมค่าใช้เกินเข้า upcoming invoice

---

## 2) ภาพรวมสถาปัตยกรรมเชิงธุรกิจ

1. สร้าง subscription
2. สร้าง invoice ตาม subscription
3. เก็บเงินผ่าน payment provider
4. ถ้าจ่ายสำเร็จ: activate subscription และออก receipt
5. สร้าง upcoming invoice สำหรับรอบหน้า
6. งาน cron รายวัน/รายเดือนจัดการ expiry, renewal, overage

---

## 3) State Model

> หมายเหตุ: state machine
> - `subscription.status`
> - `subscription.paymentStatus`
> - `invoice.status`

## 3.1 Subscription Status

- `FUTURE`: สมัคร/เปลี่ยนแผนแล้ว แต่ยังไม่ active
- `TRIAL`: ช่วงทดลอง
- `ACTIVE`: ใช้งานได้ปกติ
- `SCHEDULED`: รอเปิดรอบใหม่แต่ยังค้างชำระ
- `SUSPENDED`: ถูกพักใช้งานเพราะค้างชำระ
- `EXPIRED`: จบรอบเดือนเดิมแล้ว
- `CANCELLED`: ถูกยกเลิกโดย flow ธุรกิจ (ex. โดยแอดมิน)

## 3.2 Payment Status

- `FUTURE`: ยังไม่ถึงเวลาชำระ
- `PENDING`: ถึงเวลาชำระ/ยังไม่จ่าย
- `PAID`: จ่ายแล้ว
- `PARTIALLY_PAID`: จ่ายบางส่วน (กรณี add-on)
- `FAILED`: ชำระล้มเหลว
- `CANCELLED`: ยกเลิกการชำระ

## 3.3 Invoice Status

- `UPCOMING`: บิลรอบถัดไป (ยังไม่ถึงกำหนด)
- `ISSUED`: บิลที่ต้องชำระแล้ว
- `PAID`: ชำระแล้ว
- `PARTIALLY_PAID`: ชำระบางส่วน
- `CANCELLED`: ยกเลิกบิล

---

## 4) Flow Subscription

## 4.1 สมัครแพ็กเกจใหม่

1. ระบบสร้าง subscription โดยมี
- `subscription.status = FUTURE`
- `subscription.paymentStatus = PENDING`
2. ระบบสร้าง initial invoice โดยมี `invoice.status = ISSUED`

ผลลัพธ์: ลูกค้าเห็นบิลที่ต้องจ่ายทันที

## 4.2 สร้าง payment intent

1. ระบบค้นหา invoice `ISSUED`
2. สร้าง payment intent จาก provider
3. ผูก `transactionId` ลง invoice

ผลลัพธ์: พร้อมให้ลูกค้าทำขั้นตอนจ่ายเงิน

## 4.3 ยืนยันการชำระ

1. ระบบ retrieve payment intent
2. ถ้าสำเร็จ:
- `invoice.status -> PAID`
- `subscription.paymentStatus -> PAID`
- `subscription.status -> ACTIVE`
- stamp usability/features
- create receipt
- create upcoming invoice รอบถัดไป

ผลลัพธ์: ลูกค้าใช้งานได้ + มีหลักฐานชำระเงิน + มีบิลรอบหน้า

---

## 5) การคิดราคาและ Proration

## 5.1 Proration Discount (รายวัน)

ระบบคำนวณจากสัดส่วนวันที่เหลือในเดือน:

- `totalDays = วันทั้งหมดในเดือน`
- `daysLeft = วันคงเหลือรวมวันปัจจุบัน`
- `ratio = daysLeft / totalDays`

สำหรับแต่ละ line item ประเภท `PLAN` หรือ `ADD_ON`:

- `proratedUnitAmount = unitAmount * ratio`
- `discountPerUnit = unitAmount - proratedUnitAmount`
- `discountTotal = discountPerUnit * quantity`

จากนั้นเพิ่ม line item ประเภท `DISCOUNT`

### ตัวอย่างคำนวณ Proration

สมมติวันนี้คือวันที่ 16 ของเดือนที่มี 30 วัน

- `totalDays = 30`
- `daysLeft = 15` (นับวันที่ 16 ถึง 30)
- `ratio = 15/30 = 0.5`

ตัวอย่าง 1: Plan ราคา 1,000 บาท/เดือน

- `proratedUnitAmount = 1,000 * 0.5 = 500`
- `discountPerUnit = 1,000 - 500 = 500`
- `discountTotal = 500 * 1 = 500`

ผลที่เกิดใน invoice:

- มี line item `PLAN` = `+1,000`
- มี line item `DISCOUNT (DAILY_PRORATE)` = `-500`
- net ของ plan = `500` ก่อนภาษี

ตัวอย่าง 2: Add-on ราคา 200 บาท/หน่วย, ซื้อ 3 หน่วย

- `unitAmount = 200`, `quantity = 3`
- `proratedUnitAmount = 200 * 0.5 = 100`
- `discountPerUnit = 200 - 100 = 100`
- `discountTotal = 100 * 3 = 300`

ผลที่เกิดใน invoice:

- มี line item `ADD_ON` = `+600`
- มี line item `DISCOUNT (DAILY_PRORATE)` = `-300`
- net ของ add-on = `300` ก่อนภาษี

ตัวอย่าง 3: รวม plan + add-on และคิด VAT 7%

- ยอดก่อนส่วนลด: `1,000 + 600 = 1,600`
- ส่วนลด prorate รวม: `500 + 300 = 800`
- `subtotalWithDiscount = 1,600 - 800 = 800`
- `taxAmount = 800 * 0.07 = 56`
- `grandTotal = 800 + 56 = 856`

สรุป:

- ลูกค้าใช้ครึ่งเดือน ก็จ่ายเฉพาะส่วนที่ใช้จริงโดยประมาณ

### ตัวอย่างคำนวณ Proration

สมมติวันนี้คือวันที่ 19 ของเดือนที่มี 31 วัน

- `totalDays = 31`
- `daysLeft = 13` (นับวันที่ 19 ถึง 31)
- `ratio = 13/31 = 0.419354...`

ตัวอย่าง: Plan ราคา 1,999 บาท/เดือน

1. `proratedUnitAmount = 1,999 * (13/31) = 838.2903...`
2. ปัดเป็น 2 ตำแหน่ง = `838.29`
3. `discountPerUnit = 1,999 - 838.29 = 1,160.71`
4. `discountTotal = 1,160.71 * 1 = 1,160.71`

ผลที่เกิดใน invoice:

- PLAN = `+1,999.00`
- DISCOUNT (DAILY_PRORATE) = `-1,160.71`
- รวมก่อน VAT = `838.29`
- VAT 7% = `58.68` (จาก `838.29 * 0.07 = 58.6803`)
- Grand Total = `896.97`

## 5.2 Old Invoice Credit ตอน Upgrade

กรณี upgrade แผน ระบบเพิ่มเครดิตจากบิลเก่าตาม period ที่ยังไม่ใช้:

- ใช้ยอดรวม line item ที่เกี่ยวข้องของบิลเดิม
- คูณสัดส่วนวันที่ยังไม่ใช้
- สร้าง `DISCOUNT` line ใหม่ (`OLD_INVOICE_CREDIT`) ในบิลใหม่

ผลลัพธ์: ลูกค้าไม่ถูกคิดเงินซ้ำช่วงเวลาที่ยังไม่ใช้ของแพ็กเกจเก่า

> หมายเหตุ: ตัวอย่างคำนวณ Proration By Scenario เพิ่มเติมหัวข้อ 7.1

## 6.3 Proration By Scenario

| Scenario | DAILY_PRORATE                                 | OLD_INVOICE_CREDIT | USAGE_OVERAGE          | สรุปธุรกิจ |
|---|-----------------------------------------------|--------------------|------------------------|---|
| A: New Subscription | ใช้ (เฉพาะกรณีสมัครกลางรอบและไม่ใช่ upcoming) | ไม่ใช้             | ไม่ใช้                 | คิดตามวันที่เหลือของเดือนปัจจุบัน |
| B: Buy Add-on ระหว่างรอบ | ใช้                                           | ไม่ใช้             | ไม่ใช้                 | add-on ใหม่โดน prorate ตามวันคงเหลือ |
| C: Upgrade Now | ใช้                                           | ใช้                | ไม่ใช้                 | มีทั้งส่วนลดตามวัน + เครดิตจากบิลเก่า |
| D: Upgrade/Downgrade Next Bill | ไม่ใช้ทันที (เพราะเป็น upcoming ของรอบหน้า)   | ไม่ใช้ทันที        | ไม่ใช้                 | จะคิดเต็มรอบในงวดที่จะเริ่มจริง |
| E: Monthly Renewal Success | ไม่ใช้ (รอบใหม่เต็มเดือน)                     | ไม่ใช้             | อาจมี ถ้าใช้เกิน quota | ปกติคิดเต็มงวด + overage ถ้ามี |
| F: Monthly Renewal Failed | เหมือน E ฝั่งโครงราคา                         | ไม่ใช้             | อาจมี ถ้าใช้เกิน quota | ค้างชำระก่อน แล้วจ่ายย้อนหลังเพื่อกลับมา active |
| G: Overage | ไม่เกี่ยวกับ DAILY_PRORATE โดยตรง             | ไม่ใช้             | ใช้                    | คิดจาก usage เกิน limit ตาม unit/rate |
| H: Suspended/Overdue แล้วออกบิลใหม่ | ใช้ (ในบิลใหม่) | ไม่ใช้ | แล้วแต่ usage | ยกเลิกบิลเก่าแล้วคิดใหม่ตามวันเพื่อความยุติธรรม |

---

## 7) Scenario

## Scenario A: New Subscription (First Payment)

- สร้าง subscription ใหม่
- Expected:
- Subscription: `subscription.status: FUTURE -> ACTIVE`
- Subscription Payment: `subscription.paymentStatus: PENDING -> PAID`
- Invoice: `invoice.status: ISSUED -> PAID`
- Receipt: ถูกสร้าง 1 ใบ
- Next cycle: มี `UPCOMING` ใหม่
- Proration behavior:
- ถ้าสมัครกลางเดือน จะมี `DAILY_PRORATE` บน line item ที่เข้าข่าย

## Scenario B: Buy Add-on ระหว่างใช้งาน

- Expected:
- Add-on ใหม่เข้าคิวชำระ
- Subscription payment อาจเป็น `PARTIALLY_PAID`
- มี issued invoice สำหรับ add-on
- จ่ายแล้ว add-on ที่จ่ายจะ active
- Proration behavior:
- add-on ที่ซื้อเพิ่มกลางเดือนจะถูกคิด `DAILY_PRORATE` ตามจำนวนวันที่เหลือ

## Scenario C: Upgrade Plan ทันที

- Expected:
- สร้าง subscription ใหม่เพื่อแผนใหม่
- ออก issued invoice ทันที
- apply proration + old invoice credit
- เมื่อจ่ายสำเร็จ แผนใหม่ active ทันที
- Proration behavior:
- บิลใหม่มี `DAILY_PRORATE` (แพ็กเกจ/แอดออนที่เข้าข่าย)
- มี `OLD_INVOICE_CREDIT` หักมูลค่ารอบเก่าที่ยังไม่ใช้

## Scenario D: Upgrade/Downgrade รอบถัดไป (Next Bill)

- Expected:
- แผนใหม่ถูก schedule สำหรับรอบหน้า
- บิลใหม่อยู่ในฝั่ง upcoming ยังไม่กระทบรอบปัจจุบันทันที
- Proration behavior:
- โดยปกติยังไม่คิด prorate ตอนสร้าง เพราะบิลอยู่สถานะ `UPCOMING`
- เมื่อเข้ารอบเดือนใหม่จะคิดแบบรอบเต็มเดือนของ plan ใหม่

## Scenario E: Monthly Renewal สำเร็จ

- Expected:
- บิล `UPCOMING` -> `ISSUED`
- พยายาม auto-charge ถ้ามี payment method
- สำเร็จแล้วรอบใหม่ active ต่อเนื่อง
- Proration behavior:
- ปกติไม่ใช้ daily proration (เป็นรอบเต็มเดือน)
- ถ้ามี usage เกิน ระบบจะรวม `USAGE_OVERAGE` ในงวดนี้

## Scenario F: Monthly Renewal ล้มเหลว

- Trigger: auto-charge ไม่สำเร็จ
- Expected:
- `subscription.status = SCHEDULED`
- `subscription.paymentStatus = PENDING`
- ยังไม่ active จนกว่าจะจ่าย
- ถ้าพ้นเงื่อนไข expiration:
- `invoice.status = CANCELLED`
- `subscription.status = SUSPENDED`

## Scenario G: Overage

- Trigger: usage เกิน limit
- Expected:
- cron รายวันเติม `USAGE_OVERAGE` line item ใน upcoming invoice
- ยอดบิลถูก recalculate รวมภาษี
- ถ้า auto-renew ปิด อาจชำระเฉพาะ overage และยกเลิกส่วนรอบถัดไปตาม policy
- Proration behavior:
- ไม่คิดแบบ `daysLeft/totalDays`
- คิดจาก `จำนวนที่เกินจริง * overageRate` แล้วคำนวณภาษีตาม invoice

## Scenario H: ลูกค้าโดนระงับ/เลยวันแล้วอยากกลับมาใช้ทันที

- Trigger:
- ลูกค้าจ่ายช้าจน `subscription.status = SUSPENDED` หรือเลยกำหนดบิลเดิมแล้ว
- ลูกค้าต้องการกลับมาใช้ทันทีและต้องการยอดตามวันที่เหลือจริง

- ใช้วิธี:
1. ยกเลิก invoice เดิมที่ไม่ต้องการเรียกเก็บต่อ (`invoice.status = CANCELLED`)
2. สร้าง subscription ใหม่ในวันที่ลูกค้ากลับมาใช้
3. ระบบสร้าง issued invoice ใหม่ตามวันที่ปัจจุบัน
4. ลูกค้าชำระ invoice ใหม่ แล้วเปิดใช้งานทันทีตาม flow ปกติ

- Proration behavior:
- invoice ใหม่จะคำนวณ `DAILY_PRORATE` ตามวันคงเหลือ ณ วันที่สร้างใหม่
- ลูกค้าจะไม่ต้องจ่ายเต็มตามยอดบิลเก่าที่เกิดก่อนหน้านั้น

## 7.1 ตัวอย่างการคำนวณ Proration ตามแต่ละ Scenario (A-H)

สมมติฐานร่วมเพื่อให้เทียบง่าย:

- VAT = 7%
- เดือนนี้มี 30 วัน
- ทำรายการวันที่ 16 ของเดือน (`daysLeft = 15`, `ratio = 0.5`)
- แผน Starter = 1,000 บาท/เดือน
- แผน Growth = 2,000 บาท/เดือน
- Add-on Agent User = 200 บาท/หน่วย/เดือน
- Overage rate call out = 1 บาท/นาที

### A) New Subscription (กลางเดือน)

- PLAN = `+1,000`
- DAILY_PRORATE = `-500` (`1,000 * 0.5`)
- รวมก่อน VAT = `500`
- VAT = `35`
- Grand Total = `535`

### B) Buy Add-on ระหว่างรอบ (3 หน่วย)

- ADD_ON = `200 * 3 = +600`
- DAILY_PRORATE = `-300` (`600 * 0.5`)
- รวมก่อน VAT = `300`
- VAT = `21`
- Grand Total = `321`

### C) Upgrade Now (Starter -> Growth กลางเดือน)

- บิลใหม่ Growth line = `+2,000`
- DAILY_PRORATE ของ Growth = `-1,000`
- OLD_INVOICE_CREDIT สมมติจาก Starter ที่ยังไม่ใช้ = `-500`
- รวมก่อน VAT = `2,000 - 1,000 - 500 = 500`
- VAT = `35`
- Grand Total = `535`

หมายเหตุ: ตัวเลขเครดิตจริงขึ้นกับยอด line item ของบิลเก่าและช่วงวันที่ยังไม่ใช้จริง

### D) Upgrade/Downgrade Next Bill

- ตอนสร้าง schedule (ยังเป็น `UPCOMING`) ปกติยังไม่คิด daily prorate
- หากงวดหน้าคิดเต็มเดือน:
- Growth เดือนเต็ม = `2,000`
- VAT = `140`
- Grand Total = `2,140`

### E) Monthly Renewal Success

- สมมติงวดใหม่เป็น Starter เดือนเต็ม:
- PLAN = `1,000`
- VAT = `70`
- Grand Total = `1,070`
- ถ้ามี overage เพิ่ม 100 บาทในงวดเดียวกัน:
- ยอดก่อน VAT = `1,100`
- VAT = `77`
- Grand Total = `1,177`

### F) Monthly Renewal Failed แล้วมาจ่ายทีหลัง

- ยอด invoice ที่ค้างยังเป็นยอดเดิมของงวดนั้น (เช่น `1,070`)
- เมื่อจ่ายสำเร็จย้อนหลัง:
- `invoice.status -> PAID`
- `subscription.status: SCHEDULED -> ACTIVE`
- `subscription.paymentStatus: PENDING -> PAID`

### G) Overage

- สมมติใช้เกิน call out `120` นาที และ rate `1 บาท/นาที`
- USAGE_OVERAGE = `120`
- VAT = `8.40`
- Grand Total Overage = `128.40`

ถ้า invoice นั้นมี base plan ด้วย:

- Base plan (เช่น 1,000) + Overage (120) = `1,120`
- VAT = `78.40`
- Grand Total = `1,198.40`

### H) ระงับแล้วกลับมาใช้แบบคิดใหม่ตามวัน

สมมติ:

- บิลเก่าออกวันที่ 1 ทั้งเดือน Starter = `1,000` (VAT 7% = `70`, รวม `1,070`) แต่ลูกค้ายังไม่จ่าย
- วันที่ 20 ลูกค้าต้องการกลับมาใช้
- ยกเลิกบิลเก่าแล้วออกบิลใหม่

คำนวณบิลใหม่ (เดือน 30 วัน, วันที่ 20 เหลือ 11 วัน, ratio = `11/30 = 0.3667`):

- PLAN ใหม่ = `+1,000`
- DAILY_PRORATE ใหม่ = `-(1,000 - 366.67) = -633.33`
- รวมก่อน VAT = `366.67`
- VAT 7% = `25.67`
- Grand Total ใหม่ = `392.34`

## 7.2 Over-limit Policy และ Quota Enforcement

## 7.2.1 ค่า Policy เริ่มต้นต่อ Feature

| Feature | allowOverLimit (default) | exceedLimitAmount (default) | หมายเหตุ |
|---|---|---|---|
| `SMS_QUOTA` | `true` | `1000` | เกินได้ตาม policy |
| `CALL_OUT_MINUTES` | `true` | `1000` | มี logic ปรับตอน stamp usability |
| `RECORDING_STORAGE` | `true` | `5GB` | เกินได้ตาม policy |
| `SUPERVISOR_USER` | `false` | `0` | เกินไม่ได้ |
| `AGENT_USER` | `false` | `0` | เกินไม่ได้ |
| `NUMBER` | `false` | `0` | เกินไม่ได้ |

## 7.2.2 ระบบเช็ค quota

การเช็ค:

- ถ้า `allowOverLimit = false`:
- `limit = usageLimit`
- ถ้า `allowOverLimit = true`:
- `limit = usageLimit + exceedLimitAmount`
- ผ่านเงื่อนไขเมื่อ `totalUsed + requested <= limit`

1. ระบบตรวจว่าบริษัทมี subscription และอยู่สถานะ `ACTIVE`
2. ระบบเช็ก feature ที่ endpoint นั้นต้องใช้
3. ถ้าเกิน quota จะตอบ `FORBIDDEN` พร้อมข้อความ quota exceeded

## 7.2.3 ความเชื่อมโยงกับ Billing

- Quota guard: ใช้ตัดสิทธิ์การใช้งานแบบ real-time ตอนยิง API
- Billing overage: งานรายวันเติม `USAGE_OVERAGE` ลง `UPCOMING` invoice สำหรับ feature ที่เกิน policy และคิดเงินเพิ่มในรอบบิล

## 7.3 State Transition

## 7.3.1 Subscription Status Transition

| From | To (allowed) |
|---|---|
| `FUTURE` | `ACTIVE`, `CANCELLED`, `TRIAL`, `SCHEDULED` |
| `TRIAL` | `ACTIVE`, `CANCELLED`, `EXPIRED` |
| `ACTIVE` | `EXPIRED`, `CANCELLED` |
| `SCHEDULED` | `ACTIVE`, `SUSPENDED`, `CANCELLED` |

## 7.3.2 Payment Status Transition

| From | To (allowed) |
|---|---|
| `PENDING` | `PAID`, `FAILED`, `CANCELLED`, `EXPIRED` |
| `PAID` | `REFUNDED`, `PARTIALLY_PAID` |
| `PARTIALLY_PAID` | `PAID` |
| `FAILED` | `PENDING` |
| `FUTURE` | `CANCELLED`, `PENDING` |

## 7.3.3 Invoice Status Transition

| From | To (allowed) |
|---|---|
| `ISSUED` | `PAID`, `CANCELLED` |
| `UPCOMING` | `ISSUED`, `CANCELLED` |
