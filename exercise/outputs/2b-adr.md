Agent: Claude Code (CLI), model Claude Sonnet 5.5 (`claude-sonnet-5-5`), Run b (มี context)

# ADR-0002: ส่ง event "order placed" ไปยัง RabbitMQ เพื่อทำงานหลังสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-31
- **Deciders:** ไม่ได้ระบุไว้ใน context (ดู Open questions)

## Context and problem

ความต้องการ R-12 (`docs/scenario.md`) กำหนดให้หลังลูกค้าสั่งซื้อต้องมี 3 สิ่งใหม่: (1) ส่งอีเมลยืนยันผ่าน **MailFast** ซึ่งแทน logic mock ของ `emailservice` (2) ให้แต้มสะสมผ่าน `loyaltyservice` ที่ต้องสร้างใหม่ (3) บันทึก analytics event "order placed" และฝ่ายการตลาดคาดว่าปีหน้าจะมี action เพิ่ม (SMS, fraud scoring, อัปเดต recommendations)

**วันนี้ทำงานอย่างไร** (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`): `checkoutservice` เรียก `chargeCard` (`paymentservice`), `shipOrder` (`shippingservice`) และ `emptyUserCart` แล้วเรียก `sendOrderConfirmation` (`emailservice`) ทั้งหมดเป็น **gRPC แบบ synchronous** ตามลำดับ ถ้าส่งอีเมลล้มเหลว โค้ดแค่ `log.Warnf("failed to send order confirmation...")` แล้วยังคืนออเดอร์สำเร็จ `checkoutservice` ถือ gRPC connection และ environment variable `*_SERVICE_ADDR` อย่างละหนึ่งต่อหนึ่ง service ปลายทาง (`emailSvcAddr`, `paymentSvcAddr` ฯลฯ) ดังนั้นทุก action ใหม่จะเพิ่ม connection และการเรียกอีกหนึ่งชุดใน `PlaceOrder`

**ข้อเท็จจริงที่จำกัดการตัดสินใจ** (`docs/scenario.md` ถือเป็นข้อเท็จจริง):

| หัวข้อ | ข้อเท็จจริง | ความหมายในเรื่องนี้ |
|---|---|---|
| Latency ของ checkout | p95 ของ `PlaceOrder` ≤ **500 ms** ปัจจุบัน ≈ **300 ms** | เหลือที่ว่างราว 200 ms สำหรับทุกอย่างที่เพิ่มเข้ามา |
| MailFast | p95 **1.5 s**, p99 **8 s**, ล้มเหลว ≈ **0.5 %**, rate limit **50 emails/s** | แค่เรียก MailFast แบบ synchronous ก็เกินงบ 500 ms ที่ p95 แล้ว |
| โหลดช่วง peak | **200 orders/s** (เทศกาล 11.11) | ออเดอร์เข้ามาเร็วกว่าที่ MailFast รับอีเมลได้ 4 เท่า (200 เทียบกับ 50 ต่อวินาที) อีเมลจึงต้องเข้าคิวช่วง peak |
| แต้มสะสม | ภายใน **1 นาที** และ **exactly once** | ต้อง retry และกันซ้ำ แต่ทำแบบ asynchronous ได้ |
| Availability | ออเดอร์ต้อง **ไม่ล้ม** เพราะอีเมล แต้ม หรือ analytics ล่ม | ต้องแยก action เหล่านี้ออกจากเส้นทางของออเดอร์ |
| ทีม | นักพัฒนา 4 คน เก่ง Go และ gRPC สองคนเคยใช้ **RabbitMQ** | |
| Operations | ทุกอย่างรันบน Kubernetes ชอบ **open-source แบบ self-hosted** มากกว่า managed ที่ใช้ได้บน cloud เดียว | |

ปัญหาคือ `checkoutservice` ควรเรียก action เหล่านี้อย่างไร เพื่อปกป้อง latency และ availability ของ checkout ให้แต้มแบบ exactly once ภายใน 1 นาที และเพิ่ม action ของปีหน้าได้ในต้นทุนต่ำ

## Decision drivers

1. **Availability:** ออเดอร์ไม่ล้มเพราะอีเมล แต้ม หรือ analytics ล่ม
2. **Performance:** p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s ทั้งที่ MailFast p95 อยู่ที่ 1.5 s และ p99 ที่ 8 s
3. **Reliability / ความถูกต้อง:** แต้ม exactly once ภายใน 1 นาที และรับมือกับความล้มเหลว ~0.5 % และ limit 50 emails/s ของ MailFast โดยไม่ทำอีเมลหาย
4. **Modifiability:** เพิ่ม SMS, fraud scoring และ recommendations โดยแก้ `checkoutservice` น้อยที่สุดหรือไม่แก้เลย
5. **Operability และ cost:** เหมาะกับทักษะทีม (Go, gRPC, RabbitMQ บางส่วน) รันบน Kubernetes เป็น open-source และ self-hosted

## Considered options

1. **เรียก gRPC แบบ synchronous** จาก `checkoutservice` ไปยัง MailFast/`emailservice`, `loyaltyservice` และ analytics ต่อยอดรูปแบบปัจจุบัน
2. **เรียกแบบ asynchronous ในโปรเซสเดียว:** `checkoutservice` เปิด goroutine (fire-and-forget) เรียกปลายทางเดียวกันหลังออเดอร์สำเร็จ
3. **ส่ง event ไป RabbitMQ:** `checkoutservice` ส่ง event `OrderPlaced` หนึ่งครั้ง ส่วน `emailservice` (เรียก MailFast), `loyaltyservice` และ analytics consumer มีคิวของตัวเอง
4. **Event streaming ด้วย Kafka** (self-hosted): รูปแบบ event เดียวกัน แต่ใช้ broker แบบ log

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. gRPC แบบ synchronous | ง่ายที่สุด เป็นรูปแบบที่ทีมใช้อยู่แล้ว ไม่มี component ใหม่ | MailFast p95 1.5 s ทำให้เกินงบ p95 500 ms; p99 8 s กระทบผู้ใช้; ที่ 200 orders/s ชน limit 50/s ของ MailFast จนการเรียกล้มเหลว; ทุก action ใหม่เพิ่ม connection และการเรียกใน `PlaceOrder` (modifiability ต่ำ); latency เป็นผลรวมของทุกการเรียก |
| 2. goroutine ในโปรเซส | ไม่มี component ใหม่; เอา latency ออกจากเส้นทางตอบกลับ; เหมาะกับ Go | งานหายถ้า pod restart หรือ crash; ไม่มีบัฟเฟอร์เมื่อ 200 orders/s เกิน 50 emails/s จึงกินหน่วยความจำหรือทิ้งงาน; ไม่มี retry แบบคงทน จึงรับรอง "exactly once ภายใน 1 นาที" ของแต้มไม่ได้; ทุก action ใหม่ยังต้องแก้โค้ดใน `checkoutservice` |
| 3. RabbitMQ pub/sub | แยก `PlaceOrder` ออกจากทุก action เหลือแค่ publish ครั้งเดียว; คิวแบบ durable รับความต่าง 200 กับ 50 ต่อวินาทีและทนต่อ consumer ล่ม; retry และ dead-letter queue รับมือความล้มเหลว ~0.5 % ของ MailFast; consumer ใหม่ (SMS, fraud, recommendations) ผูกคิวของตัวเองได้โดยไม่แก้ผู้ส่ง; open-source, self-hosted, รันบน Kubernetes ได้; สองคนในทีมเคยใช้ | เป็น component แบบมี state ที่ต้องดูแล (เป็น single point of failure ถ้าไม่ทำ cluster); การส่งแบบ at-least-once อาจซ้ำ จึงต้องให้ `loyaltyservice` เป็น idempotent; publish อาจล้มเหลวทั้งที่ออเดอร์สำเร็จแล้ว (dual-write problem); eventual consistency; ติดตามข้ามหลาย service ยากขึ้น |
| 4. Kafka | log ที่คงทน replay event ได้ throughput สูง เหมาะกับ analytics | scenario ไม่ได้ระบุว่าทีมมีประสบการณ์ Kafka (ระบุแต่ RabbitMQ); ดูแลหนักกว่าที่ปริมาณงานต้องการ; replay ไม่ได้เป็นความต้องการที่ระบุ; ต้นทุนและการเรียนรู้สูงสำหรับนักพัฒนา 4 คน |

## Decision

เลือก **ทางเลือก 3: ส่ง event `OrderPlaced` ไปยัง RabbitMQ แบบ self-hosted** และทำแต่ละ action หลังสั่งซื้อเป็น consumer อิสระที่มีคิว durable ของตัวเอง

- `checkoutservice` ส่ง event ณ จุดที่ตอนนี้เรียก `sendOrderConfirmation` หลังชำระเงินและจัดส่งสำเร็จ ถ้า publish ล้มเหลวให้ log และไม่ทำให้ออเดอร์ล้ม เหมือนที่อีเมลเป็นอยู่วันนี้
- event มี `order_id` ที่ `PlaceOrder` สร้างไว้แล้ว ผู้ใช้ อีเมล และรายการสินค้า
- `emailservice` รับจากคิวของตัวเองแล้วเรียก MailFast โดย **จำกัดตัวเองไม่เกิน 50 emails/s** และ retry เมื่อล้มเหลวด้วย backoff และ dead-letter queue
- `loyaltyservice` รับจากคิวของตัวเองและ **กันซ้ำด้วย `order_id`** เพื่อให้การส่งซ้ำไม่ทำให้แต้มซ้ำ
- analytics consumer บันทึก event "order placed"
- SMS, fraud scoring และ recommendations ของปีหน้าจะเป็น consumer ใหม่ที่ผูกกับ event เดียวกัน

เหตุผลที่ตอบ driver ได้ดีที่สุด: เป็นทางเลือกเดียวที่เอา latency 1.5 s/8 s ของ MailFast ออกจาก `PlaceOrder` (performance) ทำให้ออเดอร์สำเร็จต่อเมื่อ consumer ใดล่ม (availability) รับความต่าง 200 กับ 50 ต่อวินาทีด้วย retry แบบคงทน (reliability) และเพิ่ม action ได้โดยไม่แตะผู้ส่ง (modifiability) ตรงกับทักษะทีมและความชอบ open-source/self-hosted บน Kubernetes (operability) ซึ่ง Kafka ไม่ตรง ส่วนทางเลือก 2 ถูกปฏิเสธเพราะไม่มี durability และไม่มีบัฟเฟอร์

## Consequences

- **Positive:** p95 ของ `PlaceOrder` อยู่ใกล้ ≈ 300 ms ในปัจจุบัน บวกการ publish หนึ่งครั้ง อยู่ในงบ 500 ms (quality attribute: performance)
- **Positive:** อีเมล แต้ม หรือ analytics ล่มแล้วออเดอร์ไม่ล้ม และงานที่ค้างในคิวจะถูกส่งเมื่อระบบกลับมา (availability)
- **Positive:** ให้แต้มภายใน 1 นาทีได้ด้วย retry และได้ exactly once ด้วยการจัดการแบบ idempotent (reliability)
- **Positive:** เพิ่ม action หลังสั่งซื้อใหม่เป็น consumer โดยไม่แก้ `checkoutservice` (modifiability)
- **Negative:** RabbitMQ เป็น component แบบมี state ที่ต้อง deploy, monitor และ backup (operability, cost)
- **Negative:** อีเมลและแต้มเป็น eventual consistency และช่วง peak คิวอีเมลโตเพราะ MailFast รับ 50/s เทียบกับ 200 orders/s (ความเร็วที่ผู้ใช้เห็นของอีเมล)
- **Negative:** การ trace และ debug ต้องข้ามหลาย service (observability)
- **Risks and mitigations:**
  - *อีเมลค้างช่วง peak:* คิวโตราว 150 ฉบับ/วินาทีขณะที่ออเดอร์เกิน 50/s ให้เฝ้าความลึกของคิวและตั้ง alert และตกลงความล่าช้าของอีเมลที่ยอมรับได้ (Open questions)
  - *event หาย* ถ้า publish ล้มเหลวหลังชำระเงินและจัดส่งสำเร็จ: ใช้ publisher confirms และ retry อาจต้องใช้ transactional outbox ซึ่งต้องมีที่เก็บข้อมูลที่ `checkoutservice` ดูเหมือนยังไม่มี (Open questions)
  - *ส่งซ้ำ:* consumer เป็น idempotent โดยใช้ `order_id` เป็น key และ ledger ของแต้มต้องบังคับ unique
  - *broker ล่ม:* รัน RabbitMQ เป็น cluster แบบ replicate ด้วยคิว durable และข้อความ persistent
  - *MailFast ล้มเหลว (≈ 0.5 %):* retry ด้วย backoff แล้วเข้า dead-letter queue และแจ้งเตือน

## Validation

- **ทดสอบ latency:** load test `PlaceOrder` ที่ 200 orders/s (เช่นด้วย Locust `loadgenerator`) และตรวจว่า p95 ≤ 500 ms
- **Fault injection:** หยุด MailFast (หรือให้คืน error), `loyaltyservice` และ analytics consumer ทีละตัว ตรวจว่าออเดอร์ยังสำเร็จและงานในคิวเสร็จหลังระบบกลับมา
- **ความถูกต้องของแต้ม:** ส่ง event `OrderPlaced` เดียวกันซ้ำหลายครั้ง ตรวจว่าได้แต้มครั้งเดียว และวัดเวลาจาก publish ถึงได้แต้มเทียบกับเกณฑ์ 1 นาที
- **Rate limit:** ตรวจว่า email consumer ไม่ส่งเกิน 50 emails/s และวัดเวลาที่คิวระบายหมดหลังจำลอง peak ของ 11.11
- **Modifiability:** เพิ่ม consumer จำลอง (เช่น SMS) และยืนยันว่าไม่ต้องแก้ `checkoutservice`
- **วันทบทวน:** ทบทวน ADR เมื่อกำหนด action แรกของปีหน้า หรือเมื่อมีขั้นตอนที่ต้องเกิดก่อนอีกขั้น (เช่น fraud scoring ก่อนให้แต้ม) ซึ่งอาจต้องใช้ orchestration

## Open questions

ข้อมูลต่อไปนี้ไม่ได้ให้ไว้ในไฟล์ context จึงไม่ได้สมมติ:

1. **ผู้ตัดสินใจ** และผู้อนุมัติ ADR นี้
2. **ช่วง peak ของ 11.11 ยาวแค่ไหน** และอีเมลยืนยันช้าได้ไม่เกินเท่าใด หากไม่มีสองค่านี้ ประเมินขนาดของ backlog (≈ 150 ฉบับ/วินาทีเกิน limit 50/s) ไม่ได้
3. **ความน่าเชื่อถือของการ publish:** ยอมให้ event `OrderPlaced` หายหลังชำระเงินและจัดส่งสำเร็จได้หรือไม่ `main.go` ไม่แสดง data store ใน `checkoutservice` การใช้ transactional outbox จึงต้องมีที่เก็บใหม่ ทีมต้องตัดสินใจ
4. **รายละเอียดของ MailFast:** API และการยืนยันตัวตน ค่าใช้จ่าย รองรับ idempotency key เพื่อไม่ให้ส่งอีเมลซ้ำตอน retry หรือไม่ และพฤติกรรมเมื่อเกิน limit 50/s
5. **กฎของแต้ม:** คำนวณอย่างไร ย้อนแต้มเมื่อคืนเงินหรือยกเลิกหรือไม่ และเก็บ ledger ของแต้มที่ไหน
6. **Analytics:** ปลายทางของ event, field ที่ส่ง และกฎความเป็นส่วนตัวของข้อมูลส่วนบุคคล เช่นอีเมล
7. **action ในอนาคต:** fraud scoring ต้องเสร็จก่อนให้แต้มหรือจัดส่งหรือไม่ (ถ้าใช่ ลำดับขั้นตอนจะเปลี่ยนและอาจทำให้การออกแบบแบบ pub/sub อย่างเดียวใช้ไม่ได้)
8. **ขนาดและ HA ของ RabbitMQ:** จำนวนโหนดใน cluster การตั้งค่า persistence และความสามารถในการดูแลของทีม (สองในสี่คนเคยใช้)
9. **ชะตาของ `emailservice`:** แก้ในที่เดิมให้เรียก MailFast หรือแทนด้วย service ใหม่ และจะยกเลิกการเรียก `EMAIL_SERVICE_ADDR` แบบ synchronous เดิมอย่างไร
