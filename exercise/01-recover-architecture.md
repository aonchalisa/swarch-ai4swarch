**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| frontend → productcatalogservice | system/online-boutique/kubernetes-manifests/frontend.yaml | ✅ | พบ PRODUCT_CATALOG_SERVICE_ADDR เป็น productcatalogservice:3550 ตรงกับที่ AI อ้าง |
| checkoutservice → paymentservice | system/online-boutique/kubernetes-manifests/checkoutservice.yaml | ✅ | พบ PAYMENT_SERVICE_ADDR เป็น paymentservice:50051 ตรงกับที่ AI อ้าง |
| checkoutservice → emailservice | system/online-boutique/kubernetes-manifests/checkoutservice.yaml | ✅ | พบ EMAIL_SERVICE_ADDR เป็น emailservice:5000 ตรงกับที่ AI อ้าง |
| recommendationservice → productcatalogservice | system/online-boutique/kubernetes-manifests/recommendationservice.yaml | ✅ | พบ PRODUCT_CATALOG_SERVICE_ADDR เป็น productcatalogservice:3550 ตรงกับที่ AI อ้าง |
| cartservice → redis-cart | system/online-boutique/kubernetes-manifests/cartservice.yaml | ✅ | พบ REDIS_ADDR เป็น redis-cart:6379 ตรงกับที่ AI อ้าง |
การตรวจทั้ง 5 รายการยืนยันเฉพาะการตั้งค่า endpoint ใน manifest ยังไม่ได้ตรวจ source code เพื่อยืนยันการเรียกใช้งานจริง

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย): จากการตรวจ dependency 5 รายการ พบว่าค่า endpoint ใน manifest ตรงกับที่ AI อ้างทั้งหมด จึงยังไม่พบข้อผิดพลาดในส่วนที่ตรวจ อย่างไรก็ตาม ยังไม่ได้ตรวจ source code เพื่อยืนยันการเรียกใช้งานจริง ส่วนภาษา implementation และลำดับการเรียกแต่ละ service ยังสรุปไม่ได้จากหลักฐานที่ใช้ใน Run B

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด

จากไฟล์ .proto เพียงอย่างเดียว รู้ได้ว่าแต่ละ service มี RPC อะไรและรับส่งข้อมูลแบบไหน แต่ไม่รู้ว่าใครเป็นผู้เรียก ใช้ภาษาอะไร หรือแยก deploy อย่างไร เมื่อเพิ่ม manifest ใน Run B จึงเห็น endpoint และ dependency ที่ตั้งค่าไว้ รวมถึง frontend และ Redis ที่ไม่ปรากฏเป็น service interface ใน .proto
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

ข้อผิดพลาดที่จะทำให้สมาชิกใหม่เข้าใจผิดมากที่สุดคือการแสดง dependency จาก config ราวกับยืนยันแล้วว่ามีการเรียกใช้งานจริง เพราะการตั้งค่า endpoint ไม่ได้พิสูจน์ว่าโค้ดเรียก service นั้นเสมอไป ใน Run B นี้ AI ระบุข้อจำกัดไว้แล้ว แต่หากดูเฉพาะแผนภาพโดยไม่อ่านข้อความประกอบ ก็อาจเข้าใจผิดว่าทุกเส้นผ่านการตรวจสอบจาก source code แล้ว
