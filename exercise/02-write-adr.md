**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้ เพราะไม่ได้รับ scenario และระบุว่าต้องถามเป้าหมายเพิ่มเติม | ใช้ checkout p95 ≤ 500 ms, พีก 200 orders/s, MailFast p95 1.5 s และ p99 8 s, failure 0.5%, limit 50 emails/s และแต้มภายใน 60 s |
| ทางเลือกที่พิจารณา (ระบุ) | Synchronous calls, durable message broker และ database job table พร้อม worker | Synchronous calls, RabbitMQ พร้อม durable handoff และ database job table พร้อม worker โดยเลือก RabbitMQ ตามประสบการณ์ทีมและข้อจำกัด self-hosted |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | เสนอ atomic deduplication สำหรับแต้ม แต่ไม่มีตัวเลข rate limit ของ MailFast จึงยังไม่ได้ออกแบบตามข้อจำกัดจริง | เสนอ durable ledger พร้อม unique business key และ transaction ป้องกันแต้มซ้ำ แยก email queue และจำกัดอัตรารวมทุก replica ไม่เกิน 50 requests/s รวม retry |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | เสนอ event schema, outbox และกลไก retry เพิ่มเติม แต่ระบุว่าเป็นข้อเสนอ ไม่อ้างว่าเป็นระบบปัจจุบัน และไม่กำหนดตัวเลขเอง | เสนอ journal/outbox, ledger, event schema และรายละเอียด RabbitMQ เพิ่มเติม โดยแยกเป็นข้อเสนอ ส่วน backlog อย่างน้อย 150 งาน/s เป็นการคำนวณภายใต้สมมติฐานหนึ่งอีเมลต่อคำสั่งซื้อ ไม่ใช่ตัวเลขที่ scenario ระบุโดยตรง |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เปลี่ยน Deciders เป็น ชาลิสา | ระบุผู้รับผิดชอบการตัดสินใจ |
| 2 | เพิ่มความชัดเจนว่าการเลือกแนวทางยังไม่ใช่การอนุมัติขึ้นใช้งานจริง และคงสถานะ Proposed | ยังต้องกำหนด durable storage, recovery และผ่านการทดสอบก่อน |
| 3 | เพิ่มเงื่อนไขให้เปรียบเทียบ RabbitMQ กับ job table โดยทั้งคู่แยก worker ได้ | ลดความลำเอียงในการเปรียบเทียบ และชี้เหตุผลที่ยังเลือก RabbitMQ ให้ชัดเจน |
