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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย: ไม่มี 500 ms, 200 orders/s, 50 emails/s หรือ p95 1.5 s เพราะไม่ได้เห็น scenario และยกตัวเลขเป้าหมายไปไว้ใน Open questions | ใช้ครบ: p95 ≤ 500 ms (ปัจจุบัน ~300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / ล้มเหลว ~0.5 % / limit 50 emails/s, แต้มภายใน 1 นาที และคำนวณว่าคิวอีเมลโตราว 150 ฉบับ/วินาทีช่วงพีก |
| ทางเลือกที่พิจารณา (ระบุ) | 3 ทาง: (1) synchronous orchestration (2) event-driven ผ่าน message broker (3) transactional outbox + relay เลือกทางที่ 2 แบบไม่ระบุยี่ห้อ broker | 4 ทาง: (1) synchronous gRPC (2) goroutine ใน `checkoutservice` (3) RabbitMQ (4) Kafka เลือก RabbitMQ โดยอ้างทักษะทีม (2 คนเคยใช้) และข้อกำหนด open-source/self-hosted |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: consumer เป็น idempotent โดยใช้ order id เป็น key ส่วน rate limit: พูดถึงแค่ "MailFast ล่มหรือ rate limit → retry + DLQ" และถามค่า rate limit ไว้ใน Open questions เพราะไม่รู้ ไม่ได้พูดถึงเกณฑ์ 1 นาทีของแต้ม | แต้ม: `loyaltyservice` dedupe ด้วย `order_id` และ ledger ต้องบังคับ unique ส่วน rate limit: email consumer จำกัดตัวเองที่ ≤ 50 emails/s พร้อม retry/backoff และ dead-letter queue แต่ยังไม่กำหนดจำนวน worker หรือที่เก็บของ ledger (อยู่ใน Open questions) |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่พบตัวเลขที่แต่ง และระบุว่าไม่ได้อ่านโค้ดจึงไม่ยืนยันโครงสร้างปัจจุบัน มีข้อเสนอเรื่อง outbox และ "managed service" ซึ่งเป็นแนวคิดของ AI เอง (และ scenario ชอบ self-hosted) | ตัวเลขทุกตัวมาจาก scenario หรือคำนวณจากมัน ส่วน publisher confirms, DLQ, replicated cluster เป็นการเลือกออกแบบของ AI และมีข้ออ้างที่เกินหลักฐานหนึ่งข้อ คือ "Nobody on the team is evidenced to know [Kafka]" ซึ่ง scenario ไม่ได้พูดถึงเลย (บอกแค่ว่าสองคนเคยใช้ RabbitMQ) |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่มข้อ **Publish reliability** ใน Decision: รอ publisher confirm และ retry แบบมีเวลาจำกัด แล้ว log และให้ออเดอร์สำเร็จ ยอมรับว่า event อาจหายได้ ติดตามจำนวน publish ที่ล้มเหลว และถ้าไม่เป็นศูนย์ใน load test หรือ production จะใช้ transactional outbox | ร่างของ AI ปล่อยเรื่อง event หายไว้ใน Open questions โดยไม่ตัดสิน ฉันเลือกไม่ใช้ outbox ตั้งแต่ต้นเพราะ `checkoutservice` ไม่มี data store ต้องเพิ่มของใหม่ แต่กำหนดเงื่อนไขชัดเจนว่าเมื่อไรจะเปลี่ยน |
| 2 | แก้ข้ออ้างที่ว่า RabbitMQ "เป็นทางเลือกเดียวที่เอา latency ของ MailFast ออกจาก `PlaceOrder`" | ไม่จริง เพราะ Option 2 (goroutine) ก็เอาออกได้ ร่างเองก็เขียนไว้ใน pros ของ Option 2 ข้อที่ต่างจริงคือ RabbitMQ เก็บงานแบบ durable ไว้ได้เมื่อ pod หรือ consumer ล่ม |
| 3 | แก้ข้อความของ Kafka จาก "Nobody on the team is evidenced to know it" เป็น "The scenario names RabbitMQ experience on the team but none with Kafka" และใส่ชื่อผู้ตัดสินใจ (Nitchakran Chaipojjana) แทน "Not given" | ข้อความเดิมเป็นข้ออ้างที่ scenario ไม่ได้ระบุ คำใหม่บอกแค่สิ่งที่ scenario ให้ไว้ ส่วนชื่อผู้ตัดสินใจ ฉันเป็นสถาปนิกที่ต้องปกป้อง ADR นี้ |

หมายเหตุ: Run a และ Run b เขียนเป็นภาษาไทยทั้งคู่ (Run b รันครั้งแรกได้ภาษาอังกฤษ จึงรันซ้ำด้วย `language: Thai`)
