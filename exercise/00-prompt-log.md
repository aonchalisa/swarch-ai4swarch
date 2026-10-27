**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** : ชาลิสา / 6632045321
**agent ที่ใช้:** : Codex ใน VS Code
**โมเดล:** GPT-6
**แพ็กเกจที่ใช้:** : ChatGPT Plus

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | ตรวจสอบกติกา | Codex / GPT-6 | Read AGENTS.md first. What are the rules for AI agents in this repository? | AGENTS.md | ใช่ | 5 | สรุปกติกาของ repository โดยไม่มีการแก้ไขไฟล์ |
| 2 | 1-A | Codex / GPT-6 | Read AGENTS.md first. Follow prompts/1-recover-architecture.md; run: A; evidence: system/online-boutique/protos/demo.proto | AGENTS.md; prompts/1-recover-architecture.md; system/online-boutique/protos/demo.proto | ใช่ ตามรายชื่อไฟล์ที่ agent รายงาน | 4 | ระบุ 9 service interfaces พร้อมข้อจำกัดของหลักฐาน ไม่สร้าง dependency ที่ไม่มีหลักฐานผู้เรียก |
| 3 | 1-B | Codex / GPT-6 | Read AGENTS.md first. Follow prompts/1-recover-architecture.md; run: B; evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/ | AGENTS.md; prompts/1-recover-architecture.md; system/online-boutique/protos/demo.proto; ไฟล์ใน system/online-boutique/kubernetes-manifests/: adservice.yaml, cartservice.yaml, checkoutservice.yaml, currencyservice.yaml, emailservice.yaml, frontend.yaml, kustomization.yaml, loadgenerator.yaml, paymentservice.yaml, productcatalogservice.yaml, recommendationservice.yaml, shippingservice.yaml | ใช่ ตามรายชื่อไฟล์ที่ agent รายงาน | 5 | ระบุ dependency จาก config พร้อมหลักฐาน ตรวจค่า endpoint ด้วยตนเอง 5 รายการแล้วตรงทั้งหมด แต่ยังไม่ได้ตรวจ source code เพื่อยืนยันการเรียกจริง |
| 4 | 2-a | Codex / GPT-6 | Read AGENTS.md first. Follow prompts/2-write-adr.md; run: a; context: none; requirement: R-12; language: Thai | AGENTS.md; prompts/2-write-adr.md; docs/adr/template.md | ใช่ ไม่ได้อ่าน scenario หรือไฟล์ระบบเพิ่มเติม | 4 | เปรียบเทียบสามทางเลือกและเสนอใช้ OrderPlaced โดยระบุข้อมูลที่ยังขาด ไม่กำหนดตัวเลขจาก scenario |
| 5 | 2-b | Codex / GPT-6 | Read AGENTS.md first. Follow prompts/2-write-adr.md; run: b; context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go; requirement: R-12; language: Thai | AGENTS.md; prompts/2-write-adr.md; docs/adr/template.md; docs/scenario.md; docs/adr/0001-use-redis-for-cart-storage.md; system/online-boutique/src/checkoutservice/main.go | ใช่ ตามรายชื่อไฟล์ที่ agent รายงาน | 5 | ใช้ตัวเลขและข้อจำกัดจริงใน scenario เลือก RabbitMQ ตามบริบททีม และชี้ว่าต้องออกแบบ durable journal เพิ่ม |
| 6 | 3 | Codex / GPT-6 | Read AGENTS.md first. Follow prompts/3-review-adr.md; adr: docs/adr/0002-post-order-processing.md; language: Thai | AGENTS.md; prompts/3-review-adr.md; docs/adr/0002-post-order-processing.md; docs/scenario.md | ใช่ ตามรายชื่อไฟล์ที่ agent รายงาน | 5 | ให้ผล accept with changes โดยชี้ความเสี่ยงเรื่อง durable handoff, แต้มภายใน 60 วินาทีระหว่าง outage และ backlog อีเมล ยังไม่ได้แก้ ADR ตามผลรีวิว |
| 7 | | | | | | | |