Agent: Codex (GPT-6)

# ทบทวน ADR-0002

## Quality-attribute scenarios

อ้างอิงเป้าหมายจาก `docs/scenario.md`: Quality goals and constraints; ข้อเสนอจาก `docs/adr/0002-post-order-processing.md` (ต่อไปเรียก ADR)

| Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| ลูกค้า | สั่งซื้อ 200 คำสั่งซื้อ/วินาที | พีก 11.11; MailFast p95 1.5 วินาที, p99 8 วินาที หรือล่ม | checkout บันทึกงานที่กู้คืนได้และตอบโดยไม่รออีเมล | `PlaceOrder` p95 ≤ 500 ms; ไม่มีคำสั่งซื้อล้มเหลวเพราะ MailFast; ไม่มีงานหลังคำสั่งซื้อสำเร็จสูญหาย |
| การส่งซ้ำและการล่มของผู้รับ | ส่งเหตุการณ์คำสั่งซื้อเดียวซ้ำพร้อมกัน และล่มหลัง commit ก่อน acknowledgement | พีก 200 คำสั่งซื้อ/วินาที; ledger ใช้งานได้ | `loyaltyservice` คืนสถานะเดิมโดยไม่เพิ่มแต้มซ้ำ และกู้คืนงาน | แต้มเพิ่มครั้งเดียวต่อคำสั่งซื้อภายใน 60 วินาทีจากเวลาสั่งซื้อ รวมเวลาค้างทุกขั้น; ต้องทดสอบ outage ของ loyalty แยกและรายงานการผิดเส้นตาย |
| งานอีเมลและ MailFast | สร้างหนึ่งอีเมลต่อคำสั่งซื้อ พร้อมความล้มเหลวประมาณ 0.5% | พีก 200 คำสั่งซื้อ/วินาที; quota 50 อีเมล/วินาที | จำกัดการส่งรวมทุก replica และแยก backlog จากงานแต้ม | ไม่เกิน quota; [Inferred] backlog เพิ่มอย่างน้อย 150 งาน/วินาทีก่อน retry; เส้นตายอีเมลยังไม่มี ต้องตกลงก่อนตัดสินว่าความจุเพียงพอ |

## Findings table

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | ADR: Decision แยกคิวอีเมล แต้ม และ analytics; มี rate limiter รวมและ ledger ที่บันทึกแต้มกับ deduplication แบบ atomic | [Inferred] แนวทางเหมาะกับ MailFast ที่ช้าและจำกัด quota และป้องกันแต้มซ้ำจาก message redelivery; ยังต้องพิสูจน์ด้วยการทดสอบ |
| Risk | ADR: Decision เลื่อนการเลือก journal และจุดยืนยันคำสั่งซื้อไปออกแบบภายหลัง | ยังพิสูจน์ไม่ได้ว่าล่มหลัง payment/shipping แล้วทุกคำสั่งซื้อสำเร็จมีงานที่กู้คืนได้; publisher confirms ไม่ปิดช่องว่างก่อนบันทึก journal |
| Risk | ADR: Consequences ยอมรับว่า outage ของ loyalty ยาวเกินเส้นตายทำให้แต้มไม่ทัน แต่ยังไม่มีแนวทางที่ธุรกิจอนุมัติ | `docs/scenario.md`: Loyalty points กำหนดภายใน 1 นาที ขณะที่ Availability ห้ามให้งานปลายทางล่มทำให้คำสั่งซื้อล้มเหลว; ข้อเสนอจึงยังไม่ผ่านข้อกำหนดทั้งหมด |
| Sensitivity point | ระยะพีก อัตราหลังพีก และ quota MailFast กำหนดอายุงานและขนาดคิว (ADR: Open questions) | [Inferred] พีกยาว T วินาทีสร้าง backlog อย่างน้อย 150T งาน; ถ้าอัตรางานใหม่รวม retry ไม่ต่ำกว่ากำลังส่งจริง คิวจะไม่ระบาย |
| Trade-off point | durable journal และ ledger เพิ่มความทนทาน แต่เพิ่ม I/O ใน checkout และระบบ stateful ที่ต้องดูแล (ADR: Decision, Consequences) | [Inferred] reliability เพิ่ม แลกกับ latency และภาระทีม 4 คน; ต้องวัด p95 รวม durable handoff และเปรียบเทียบต้นทุนกับ job table |

## Missing or weak

- **ตัวเลือก:** มีสามทางเลือกที่ต่างกันแล้ว แต่ข้อเสียเรื่อง worker กลางของทางเลือก 3 ไม่จำเป็นต้องเกิดเสมอ เพราะแยก worker ตามประเภทงานได้เช่นกัน ควรเปรียบเทียบ job table กับ RabbitMQ โดยใช้ความทนทานและการแยก worker เท่ากัน โดยเฉพาะเมื่อทางเลือก 2 ต้องเพิ่มฐานข้อมูลอยู่แล้ว (ADR: Pros and cons, Decision)
- **ผลกระทบ:** ADR กล่าวถึง broker replication และความจุ แต่ยังไม่เลือก queue/replication topology, พื้นที่เก็บ, retention, เกณฑ์แจ้งเตือน และพฤติกรรมเมื่อคิวหรือ journal เต็ม ซึ่งอาจทำให้ outage อีเมลลามมาปิดกั้น checkout (ADR: Consequences, Open questions)
- **คำกล่าวที่ยังไม่ยืนยัน:** การมี RabbitMQ และ outbox ยังไม่พิสูจน์ no-loss, p95 ≤ 500 ms หรือแต้มทัน 60 วินาที; ADR ระบุเงื่อนไขไว้ถูกต้องแต่ต้องมีผลทดสอบก่อนรับรอง ข้ออ้างรายละเอียด `main.go` และ ADR-0001 ตรวจตรงกับต้นฉบับไม่ได้ในการทบทวนนี้ เพราะไม่ได้รับอนุญาตให้อ่าน; หลักฐานอิสระในขอบเขตนี้เป็น **Not evidenced in the provided files** นอกเหนือจากพฤติกรรม synchronous และ warning ที่ `docs/scenario.md`: How it works today ยืนยัน

## Verdict

**accept with changes** — เห็นชอบทิศทาง แต่ให้คงสถานะ `Proposed` จนปิดสามประเด็นต่อไปนี้

1. ระบุ durable journal, จุดยืนยันคำสั่งซื้อ, stable identity และ recovery state ระหว่าง payment/shipping/journal พร้อมผล fault injection และทดสอบ p95 ≤ 500 ms ที่ 200 คำสั่งซื้อ/วินาที
2. ให้ธุรกิจยืนยันวิธีจัดการความขัดแย้งระหว่างแต้มภายใน 60 วินาทีกับ outage; ระบุฐานข้อมูล ledger และทดสอบ atomic deduplication เมื่อส่งซ้ำ/ล่ม โดยไม่ลดข้อกำหนดเอง
3. ตกลงเส้นตายอีเมลและระยะพีก แล้วกำหนดความจุคิว การระบาย backlog และนโยบายเมื่อเต็ม พร้อมผู้รับผิดชอบปฏิบัติการ; หาก quota 50 ไม่พอ ต้องตกลงเพิ่ม quota หรือเปลี่ยนเงื่อนไขกับธุรกิจ

ไฟล์ที่อ่าน: `AGENTS.md`, `prompts/3-review-adr.md`, `docs/adr/0002-post-order-processing.md`, `docs/scenario.md`
