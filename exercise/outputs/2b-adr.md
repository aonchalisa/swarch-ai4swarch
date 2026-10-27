Agent: Codex (GPT-6)

# ADR-0002: แยกงานหลังสั่งซื้อผ่าน RabbitMQ และผู้ประมวลผลอิสระ

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ยังไม่ระบุ ต้องให้ผู้รับผิดชอบระบบและธุรกิจยืนยัน

## Context and problem

R-12 กำหนดให้ส่งอีเมลผ่าน MailFast แทน logic จำลองของ `emailservice`, ให้แต้มผ่าน `loyaltyservice` ใหม่ และบันทึก analytics หลังสั่งซื้อ โดยจะเพิ่ม SMS, fraud scoring และการอัปเดต recommendations ในปีหน้า (`docs/scenario.md`: The new requirement)

`system/online-boutique/src/checkoutservice/main.go` ใน `PlaceOrder` เรียก `chargeCard` → `shipOrder` → `emptyUserCart` → สร้าง `orderResult` → `sendOrderConfirmation` ก่อนคืน response; หากอีเมลผิดพลาดจะเรียก `log.Warnf` และยังคืนผลสำเร็จ ส่วน `sendOrderConfirmation` เรียก `EmailServiceClient.SendOrderConfirmation` ผ่าน gRPC แบบ synchronous

| ข้อเท็จจริง | หลักฐาน |
|---|---|
| `PlaceOrder` ต้องมี p95 ≤ 500 ms; ปัจจุบันประมาณ 300 ms | `docs/scenario.md`: Checkout latency |
| พีก 200 คำสั่งซื้อ/วินาที | `docs/scenario.md`: Peak load |
| MailFast p95 1.5 วินาที, p99 8 วินาที, ล้มเหลวประมาณ 0.5%, จำกัด 50 อีเมล/วินาที | `docs/scenario.md`: MailFast |
| แต้มต้องเข้าภายใน 1 นาทีและห้ามได้ซ้ำ; คำสั่งซื้อต้องไม่ล้มเหลวเพราะอีเมล แต้ม หรือ analytics ล่ม | `docs/scenario.md`: Loyalty points, Availability |
| ทีม 4 คนถนัด Go/gRPC; 2 คนเคยใช้ RabbitMQ; ใช้ Kubernetes และต้องการ open-source แบบ self-hosted | `docs/scenario.md`: Team, Operations |

[Inferred] ไม่ควรรอ MailFast ในเส้นทาง checkout เพราะ latency ของผู้ให้บริการเสี่ยงเกินเป้าหมาย ทั้งนี้ไม่สามารถบวกค่า p95 ของแต่ละระบบเป็น p95 รวมได้โดยตรง หากหนึ่งคำสั่งซื้อส่งหนึ่งอีเมล พีกจะสร้างงาน 200 งาน/วินาที เทียบกับความสามารถส่งสูงสุด 50 งาน/วินาที ทำให้ backlog เพิ่มอย่างน้อย 150 งาน/วินาทีก่อนนับ retry; คิวช่วยรองรับช่วงพีก แต่ไม่เพิ่มขีดจำกัดผู้ให้บริการ

ไฟล์ `main.go` ที่ให้มาแสดง `orderID` ที่สร้างใหม่ต่อคำขอและ `orderResult` ในหน่วยความจำ แต่ไม่แสดงการบันทึกคำสั่งซื้อหรือ outbox แบบ durable; การมีฐานข้อมูลคำสั่งซื้อและขอบเขต transaction เป็น **Not evidenced in the provided files** จึงห้ามสมมติว่าเพิ่ม outbox ลง transaction เดิมได้ทันที ส่วน `docs/adr/0001-use-redis-for-cart-storage.md` ยอมรับ durability ต่ำสำหรับตะกร้าเพราะยังไม่เกี่ยวกับเงิน เหตุผลนี้ใช้แทนความทนทานของงานหลังชำระเงินไม่ได้

## Decision drivers

1. **Availability:** อีเมล แต้ม และ analytics ล่มต้องไม่ทำให้คำสั่งซื้อล้มเหลว (`docs/scenario.md`: Availability)
2. **Correctness และ timeliness:** แต้มครั้งเดียวภายใน 60 วินาที (`docs/scenario.md`: Loyalty points)
3. **Performance:** checkout p95 ≤ 500 ms ที่พีก 200 คำสั่งซื้อ/วินาที (`docs/scenario.md`: Checkout latency, Peak load)
4. **Modifiability:** เพิ่มผู้รับงานใหม่ตามแผนการตลาด โดยลดการแก้ checkout (`docs/scenario.md`: R-12)
5. **Operability และ cost:** เหมาะกับทีมขนาดเล็ก ประสบการณ์ RabbitMQ และ Kubernetes แบบ self-hosted (`docs/scenario.md`: Team, Operations)

## Considered options

1. เพิ่มการเรียก synchronous จาก checkout ไปยัง `emailservice`/MailFast, `loyaltyservice` และ analytics โดยกำหนด timeout และไม่คืนความล้มเหลวของงานเหล่านี้เป็นความล้มเหลวคำสั่งซื้อ
2. เผยแพร่ `OrderPlaced` ผ่าน RabbitMQ โดยมี durable handoff และผู้รับแยกตามงาน
3. ใช้ durable job table และ worker สำรวจงานจากฐานข้อมูล แยกสถานะอีเมล แต้ม และ analytics โดยไม่ใช้ broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1 | ใช้ทักษะ Go/gRPC เดิม; ลำดับเรียกเข้าใจง่าย | [Inferred] รอ MailFast เสี่ยงผิดเป้าหมาย latency; timeout สั้นอาจทิ้งงาน; ยังต้องมี durable retry และ rate limiting; เพิ่มงานแล้ว checkout ต้องเปลี่ยน |
| 2 | [Inferred] แยกความล้มเหลวและกำลังประมวลผลแต่ละงาน; เพิ่มผู้รับเหตุการณ์ได้; สอดคล้องประสบการณ์ RabbitMQ ของทีม | เพิ่มระบบ stateful และงานปฏิบัติการ; ต้องจัดการข้อความซ้ำและ publish gap; ไม่แก้เพดาน MailFast หรือรับประกันแต้มทันเวลาระหว่าง outage ยาว |
| 3 | [Inferred] เก็บสถานะงานและ retry ชัดเจน; ไม่ต้องดูแล broker | ยังต้องเพิ่มฐานข้อมูล durable ที่ไม่มีหลักฐานว่ามีอยู่; polling เพิ่มเวลารอและภาระ; ต้องจัดการ locking และการกระจายงาน; worker กลางเสี่ยง coupling เมื่อเพิ่มงาน |

## Decision

เลือกทางเลือก 2: เสนอใช้ **RabbitMQ แบบ self-hosted บน Kubernetes** พร้อมช่องรับงานอิสระสำหรับ `emailservice`, `loyaltyservice` และ analytics เพื่อแยกเวลารอ MailFast และรองรับงานใหม่ เหตุผลอ้างอิงข้อจำกัดจาก `docs/scenario.md`; รายละเอียดต่อไปนี้เป็นแบบที่เสนอ ไม่ใช่สภาพระบบปัจจุบัน

- นำ `sendOrderConfirmation` ออกจากเส้นทาง synchronous ของ `PlaceOrder` และเปลี่ยนให้ `emailservice` รับงานไปเรียก MailFast เพิ่มผู้รับงานแต้มและ analytics โดย checkout เผยแพร่เหตุการณ์เดียว ไม่เรียกงานใหม่ตามลำดับ
- เสนอ durable exchange พร้อม durable queue แยกสามงาน ใช้ persistent messages, publisher confirms และตรวจข้อความที่ไม่มี route; ใช้ manual acknowledgement หลังผู้รับบันทึกผลสำเร็จ การยืนยัน publish ไม่ได้แปลว่างานปลายทางสำเร็จ
- ใช้ `OrderPlaced` ที่มี `event_id`, `order_id`, `occurred_at`, `schema_version`, ตัวระบุลูกค้า และข้อมูลขั้นต่ำสำหรับแต่ละงาน จำกัดข้อมูลส่วนบุคคลตามสิทธิ์ผู้รับและไม่ส่งข้อมูลบัตร ไม่ผูก schema กับ `PlaceOrderRequest` ทั้งก้อน
- แยก email queue และ worker พร้อม rate limiter รวมทุก replica ไม่เกิน 50 คำขอ/วินาทีรวม retry; ใช้ timeout, backoff และ jitter โดยพิจารณา p99 8 วินาที จำนวนงานพร้อมกันและค่าจริงต้องวัดก่อนกำหนด ไม่ให้การรออีเมลกินกำลัง worker แต้ม
- ให้ `loyaltyservice` ใช้ durable ledger พร้อม unique business key เช่น `(order_id, award_type)` และเพิ่มแต้มกับบันทึกกุญแจใน transaction เดียว จึงรับข้อความซ้ำได้โดยไม่เพิ่มแต้มซ้ำ; broker ใช้ at-least-once ไม่อ้าง exactly-once delivery
- ให้ analytics ตรวจซ้ำตาม `event_id`; ตรวจสอบว่า MailFast รองรับ idempotency key หรือค้นสถานะการส่งหรือไม่ เพราะการล่มหลังส่งสำเร็จแต่ก่อนบันทึกผลอาจทำให้อีเมลซ้ำ การรับประกันไม่ซ้ำของแต้มไม่ครอบคลุมอีเมลโดยอัตโนมัติ
- retry แยกตามประเภทงานและแยกข้อความที่ประมวลผลไม่ได้ไป dead-letter queue พร้อมแจ้งเตือนและ replay; งานแต้มต้องติดตามอายุจากเวลาสั่งซื้อจริง รวมเวลาอยู่ใน outbox และคิว ไม่เริ่มนับใหม่เมื่อ retry

**จุดที่ต้องออกแบบเพิ่มก่อนยอมรับ:** เพิ่ม durable order journal/outbox และ relay ที่ส่งซ้ำได้ เพื่อให้ broker หรือผู้รับล่มแล้วงานยังอยู่ แต่ `main.go` ไม่มี transaction บันทึกคำสั่งซื้อให้ใช้ร่วมอยู่แล้ว จึงต้องกำหนดจุดยืนยันคำสั่งซื้อและ recovery ของขั้นตอน payment/shipping โดยเฉพาะช่วงล่มหลังทำผลข้างเคียงก่อนบันทึกเหตุการณ์ ห้ามใช้ goroutine ในหน่วยความจำหรือเพียง log warning แทน durable handoff และห้ามอ้างว่า outbox ทำ transaction ร่วมกับ payment/shipping ภายนอกได้

เมื่อ journal ใช้งานได้ checkout ต้องบันทึกงานได้ก่อนคืนผลสำเร็จโดยไม่รอ broker หรือผู้รับปลายทาง; การล่มของ journal เองยังต้องกำหนดนโยบาย recovery กับธุรกิจ จึงยังไม่รับรองว่าทุกกรณีความล้มเหลวผ่านข้อกำหนดแล้ว ต้องพิสูจน์ด้วย fault injection ก่อนรับรอง ADR นี้

## Consequences

- **Positive:** [Inferred] ตัดเวลารอ MailFast ออกจาก checkout และแยกความล้มเหลวของผู้รับงาน หาก durable handoff ถูกต้อง (quality attributes: performance, availability)
- **Positive:** [Inferred] เพิ่ม SMS หรือผู้รับใหม่ได้ด้วยการผูกช่องรับเหตุการณ์โดยไม่แก้เส้นทาง checkout เมื่อ schema เพียงพอ (quality attribute: modifiability)
- **Positive:** ledger ที่บันทึกกุญแจและแต้มแบบ atomic รองรับผลทางธุรกิจครั้งเดียวแม้ส่งข้อความซ้ำ (quality attribute: correctness; เป็นการออกแบบที่เสนอ ต้องทดสอบ)
- **Negative:** เพิ่ม RabbitMQ, journal/outbox, ledger และระบบติดตาม retry ซึ่งเป็นภาระสำหรับทีม 4 คน (quality attributes: operability, cost)
- **Negative:** งานหลังสั่งซื้อเป็น eventual consistency; อีเมลค้างเพิ่มอย่างน้อย 150 งาน/วินาทีระหว่างพีกตามสมมติฐานหนึ่งอีเมลต่อคำสั่งซื้อ และอาจเพิ่มมากกว่านี้จากความล้มเหลว/ retry (quality attributes: timeliness, capacity)
- **Risks and mitigations:** แต้มภายใน 1 นาทีเป็นไปไม่ได้หากระบบให้แต้มใช้งานไม่ได้เกินเส้นตาย; แยกคิว จัดกำลังประมวลผลและแจ้งเตือนตามอายุงาน พร้อมให้ธุรกิจตกลงขอบเขต outage/SLO โดยไม่ลดข้อกำหนดเอง (quality attributes: availability, timeliness)
- **Risks and mitigations:** ส่งงานสูญหายระหว่าง payment/shipping/journal หรือ publish; ใช้ durable recovery state, stable business identity, outbox relay และ reconciliation หลังตรวจความสามารถ idempotency ของบริการที่เกี่ยวข้อง (quality attribute: reliability)
- **Risks and mitigations:** broker/ฐานข้อมูลอาจกลายเป็นจุดล้มเหลว; ออกแบบ replication, persistence, backup และทดสอบ recovery ตามเป้าหมายที่ต้องตกลง ไม่ใช้ Redis ตะกร้าเป็น ledger โดยอาศัย ADR-0001 (quality attributes: durability, availability)
- **Risks and mitigations:** replay อาจทำให้ผลซ้ำหรือข้อมูลส่วนบุคคลกระจายเกินจำเป็น; ใช้ deduplication แบบ durable, สิทธิ์เข้าถึง และ retention ที่อนุมัติ (quality attributes: correctness, security)

## Validation

- ทดสอบ `PlaceOrder` ที่ 200 คำสั่งซื้อ/วินาทีและตรวจ p95 ≤ 500 ms รวมค่า durable handoff; จำลอง MailFast ตาม p95 1.5 วินาที, p99 8 วินาที, ความล้มเหลว 0.5% และจำกัด 50 คำขอ/วินาที (`docs/scenario.md`: Quality goals and constraints)
- ตรวจแต้มหนึ่งครั้งต่อคำสั่งซื้อและเสร็จภายใน 60 วินาทีที่พีก รวมเวลา journal/relay/queue; ส่งข้อความซ้ำพร้อมกันหลาย worker และทำให้ล่มระหว่าง commit กับ acknowledgement แล้วตรวจ ledger
- ปิด MailFast, loyalty และ analytics แยกกัน ตรวจว่าคำสั่งซื้อยังสำเร็จและงานไม่หาย; ทดสอบ broker ล่มโดย journal ยังทำงาน และตรวจการส่งต่อหลังฟื้น พร้อมรายงานการผิดเส้นตายแต้มตรงตามจริง
- ทำ fault injection หลัง payment, หลัง shipping, ก่อน/หลังบันทึก journal, ก่อน/หลัง publisher confirm และหลัง consumer commit เพื่อพิสูจน์ reconciliation โดยไม่ชำระเงิน จัดส่ง หรือให้แต้มซ้ำ
- วัดอายุงานแยกแต่ละคิว, outbox lag, retry, dead-letter, อัตราส่ง MailFast รวมทุก replica และเวลาระบาย backlog; ใช้ระยะพีกและอัตราหลังพีกที่ธุรกิจยืนยันเพื่อคำนวณพื้นที่เก็บและเวลารอ ห้ามอ้างว่าคิวจะระบายได้หากอัตราระยะยาวยัง ≥ 50 อีเมล/วินาที
- ทดลองเพิ่มผู้รับเหตุการณ์จำลองและทดสอบ schema compatibility; ให้ทีมที่ดูแลระบบทดสอบ replay, restore และ runbook ก่อนพิจารณาเปลี่ยนสถานะเป็น Accepted

## Open questions

- ใครเป็นผู้อนุมัติ ADR และรับผิดชอบ broker, ledger, journal และงานค้าง?
- พีก 200 คำสั่งซื้อ/วินาทียาวเท่าใด อัตราปกติเท่าใด และยอมให้อีเมล/analytics ล่าช้าได้เท่าใด? ขอเพิ่ม quota MailFast ได้หรือไม่หากความต้องการเกิน 50 อีเมล/วินาทีต่อเนื่อง?
- นิยามจุดที่คำสั่งซื้อสำเร็จคือจุดใด จะเพิ่ม durable order journal ที่ใด และจัดการ journal ล่มหรือการล่มหลังชำระเงิน/จัดส่งอย่างไร?
- API payment/shipping รองรับ stable idempotency key และการค้นผลย้อนหลังหรือไม่? `main.go` สร้าง UUID ใหม่ต่อคำขอ จึงต้องออกแบบตัวระบุที่คงเดิมสำหรับคำขอสั่งซื้อที่ retry
- ต้องใช้ฐานข้อมูลใดสำหรับ journal และ loyalty ledger; ต้องการ RPO/RTO, retention และงบประมาณเท่าใด?
- ข้อกำหนดแต้มภายใน 1 นาทีครอบคลุม outage แบบใด และธุรกิจจะจัดการกรณีที่ไม่สามารถทำตามได้อย่างไร?
- กฎคำนวณแต้ม การคืนเงิน และการยกเลิกคำสั่งซื้อมีผลอย่างไรต่อกุญแจป้องกันซ้ำและรายการชดเชย?
- MailFast ใช้ protocol/API ใด รองรับ idempotency และตรวจสถานะส่งหรือไม่; ต้องตั้ง timeout และ retry ตามข้อกำหนดใด?
- analytics บันทึกที่ใด ต้องเรียงลำดับหรือรองรับ replay แค่ไหน และ event schema ต้องมีข้อมูลส่วนบุคคลใด?

ไฟล์ที่อ่าน: `AGENTS.md`, `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `system/online-boutique/src/checkoutservice/main.go`
