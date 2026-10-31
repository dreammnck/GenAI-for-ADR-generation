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
| checkoutservice → cartservice | `checkoutservice.yaml` (`CART_SERVICE_ADDR`) | ✅ | `main.go` L113 อ่าน `CART_SERVICE_ADDR`, L325 เรียก `GetCart`, L333 เรียก `EmptyCart` |
| checkoutservice → paymentservice | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | `main.go` L116 อ่าน env, L370 เรียก `Charge` |
| checkoutservice → emailservice | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`) | ✅ | `main.go` L115 อ่าน env, L380 เรียก `SendOrderConfirmation` |
| cartservice → redis-cart | `cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`) | ✅ | `cartservice.yaml` L50–51 ตรงตามที่อ้าง และ L92, L146 เป็น Deployment/Service ของ `redis-cart` |
| frontend → cartservice | `frontend.yaml` (`CART_SERVICE_ADDR=cartservice:7070`) | ✅ | `frontend.yaml` L73–74 ตรงตามที่อ้าง |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - ใน 5 แถวที่ตรวจ ไม่พบ dependency ที่ผิด
  - ขาดเมธอดที่เรียกจริง: 1B บอกแค่ว่า "เรียก" ไม่รู้ว่าเรียก RPC ไหน
  - ขาดพฤติกรรมเมื่อล้มเหลว: `main.go` L273–277 ถ้าส่งอีเมลไม่สำเร็จแค่ `log.Warnf` แล้วออเดอร์ยังสำเร็จ แต่แผนภาพวาดลูกศร `checkout → email` เหมือนทุกเส้น
  - `shoppingassistantservice`: `frontend.yaml` ตั้ง `SHOPPING_ASSISTANT_SERVICE_ADDR` แต่ไม่มี manifest 1B ตัดออกจากแผนภาพ (ระบุไว้ใน Gaps)
  - protocol "gRPC" ในแถว 3–16 อนุมานจากฝั่งปลายทาง ไม่ใช่ฝั่งผู้เรียก

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ

1. จาก `.proto` อย่างเดียว agent บอกได้แค่ว่าแต่ละ service เสนออะไร ไม่รู้ว่าใครเรียกใคร มี frontend หรือ Redis หรือไม่ และ service ถูก deploy แยกกันจริงหรือเปล่า เพราะ proto เป็นแค่สัญญาของ interface ไม่ใช่การเชื่อมต่อตอนรันจริง ต้องดู manifest (env var `*_SERVICE_ADDR`) จึงจะเห็นความสัมพันธ์
2. แผนภาพวาด `checkoutservice → emailservice` เหมือนทุกเส้น คนใหม่จะคิดว่าอีเมลล่มแล้วสั่งซื้อไม่ได้ แต่โค้ดแค่เตือนแล้วปล่อยออเดอร์สำเร็จ (`main.go` L273–277) manifest ไม่แสดงเรื่องนี้ จึงต้องอ่านโค้ดประกอบ
