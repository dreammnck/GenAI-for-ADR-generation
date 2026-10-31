Agent: Claude Code, model Claude Sonnet 5.5 (run a, zero-shot, context: none)

# ADR-0002: ใช้ event-driven (message broker) สำหรับ post-order actions หลังสั่งซื้อ

- **Status:** Proposed
- **Date:** 2026-10-31
- **Deciders:** ยังไม่ระบุ (Not evidenced in the provided files) ดู Open questions

## Context and problem

หลังลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำ post-order actions ดังนี้ (จาก requirement):

1. ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** (external email provider ใหม่) แทน logic แบบ mock ใน `emailservice`
2. ให้ **loyalty points** ผ่าน `loyaltyservice` ใหม่ (ต้องสร้าง)
3. บันทึก **"order placed" analytics event** สำหรับทีม marketing

นอกจากนี้ marketing คาดว่าจะมี post-order actions เพิ่มในปีหน้า (SMS, fraud scoring, recommendations update)

ปัญหาที่ต้องตัดสินใจ: ใครเป็นผู้เรียก action เหล่านี้ และเรียกอย่างไร (synchronous หรือ asynchronous) โดยไม่ให้ความล้มเหลวหรือความช้าของ action เสริมกระทบการสั่งซื้อของลูกค้า และให้เพิ่ม action ใหม่ได้ง่าย

หมายเหตุ: ADR นี้เขียนจาก requirement อย่างเดียว (zero-shot) ไม่ได้อ่านโค้ดหรือเอกสารของระบบ ดังนั้นสมมติฐานเกี่ยวกับโครงสร้างปัจจุบัน (เช่น ใครเรียก `emailservice` อยู่ตอนนี้) **ไม่ได้ถูกยืนยัน** (Not evidenced in the provided files)

## Decision drivers

- **Availability / fault isolation:** MailFast, loyalty หรือ analytics ล่มต้องไม่ทำให้ลูกค้าสั่งซื้อไม่สำเร็จ
- **Modifiability:** ต้องเพิ่ม action ใหม่ (SMS, fraud scoring, recommendations update) โดยแก้ผู้สั่งซื้อให้น้อยที่สุด
- **Performance (latency ของการ checkout):** ลูกค้าไม่ควรรอ action เสริมทั้งหมด
- **Reliability / data consistency:** ต้องไม่สูญเสีย event (เช่น points ไม่ถูกให้, อีเมลไม่ถูกส่ง) และต้องกัน duplicate
- **Cost / operational complexity:** ทีมต้องดูแล infrastructure เพิ่ม
- **Security / privacy:** ข้อมูลลูกค้า (อีเมล, คำสั่งซื้อ) ถูกส่งไปผู้ให้บริการภายนอก (MailFast)

## Considered options

1. **Synchronous orchestration:** service ที่รับ order เรียก MailFast, `loyaltyservice` และ analytics ตรง ๆ ทีละตัว (หรือแบบขนาน) ภายใน flow การสั่งซื้อ
2. **Event-driven ผ่าน message broker (pub/sub):** หลัง order สำเร็จ ให้ publish event `OrderPlaced` หนึ่งครั้ง แล้วแต่ละ consumer (email, loyalty, analytics และอื่น ๆ ในอนาคต) subscribe และประมวลผลแบบ asynchronous
3. **Transactional outbox + relay ไป broker:** เหมือนข้อ 2 แต่เขียน event ลงตารางเดียวกับข้อมูล order ใน transaction เดียว แล้วมี relay ส่งต่อไป broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous orchestration | เรียบง่าย ไม่ต้องมี infrastructure ใหม่; debug ง่าย; ได้ผลลัพธ์ทันที | ผู้เรียกต้องรู้จักทุก action (coupling สูง); เพิ่ม action = แก้โค้ดผู้เรียกทุกครั้ง; latency บวกกันและ action ที่ช้า/ล่มกระทบ checkout; ต้องเขียน retry/timeout เอง |
| 2. Event-driven ผ่าน broker | ผู้ publish ไม่รู้จัก consumer (loose coupling); เพิ่ม consumer ใหม่โดยไม่แก้ผู้ publish; ความล้มเหลวถูกแยก และ retry ได้; checkout latency ต่ำ | ต้องมี broker เพิ่ม (cost/ops); eventual consistency; ต้องออกแบบ idempotency และ dead-letter; ติดตามปัญหายากขึ้น; publish อาจล้มเหลวหลัง order สำเร็จ (dual-write) |
| 3. Outbox + relay | ไม่สูญเสีย event แม้ broker ล่มชั่วคราว; ความสอดคล้องระหว่าง order กับ event ดีที่สุด | ซับซ้อนที่สุด; ต้องมี data store ที่รองรับ transaction และ relay process; ไม่ทราบว่าระบบปัจจุบันมี store ของ order หรือไม่ (Not evidenced in the provided files) |

## Decision

เลือก **ตัวเลือก 2: Event-driven ผ่าน message broker** โดยให้ service ที่ทำให้ order สำเร็จ publish event `OrderPlaced` แล้วให้ consumer แยกกันสามตัวเริ่มต้น:

- email consumer เรียก MailFast (ผ่าน adapter ที่แทน logic mock ของ `emailservice`)
- `loyaltyservice` ให้ points
- analytics consumer บันทึก "order placed" event

เหตุผล: ตัวเลือกนี้ตอบ driver สำคัญสุดสองข้อได้ดีที่สุด คือ **availability** (การล่มของ MailFast/loyalty/analytics ไม่ทำให้ checkout ล้มเหลว) และ **modifiability** (SMS, fraud scoring, recommendations update ในอนาคตเป็นเพียง consumer ใหม่) ตัวเลือก 1 เหมาะกับ 3 actions แต่ไม่รองรับการเติบโตที่ marketing คาดไว้ ส่วนตัวเลือก 3 ให้ความทนทานสูงสุด แต่ยังตัดสินใจไม่ได้จนกว่าจะทราบว่า order ถูกเก็บที่ไหน จึงเสนอให้พิจารณาเป็นขั้นต่อไปหากพบว่า event หาย (ดู Open questions)

ข้อกำหนดของการ implement: consumer ทุกตัวต้อง **idempotent** (ใช้ order id เป็น key), มี retry พร้อม backoff และ dead-letter queue, และ event ต้องมีเฉพาะข้อมูลที่จำเป็น (minimise PII)

## Consequences

- **Positive:** checkout ไม่ถูกบล็อกหรือล้มเหลวเพราะ action เสริม (quality attribute: availability, performance)
- **Positive:** เพิ่ม post-order action ใหม่ได้โดยเพิ่ม consumer ไม่ต้องแก้ผู้ publish (quality attribute: modifiability)
- **Positive:** แต่ละ consumer scale และ deploy แยกกันได้ (quality attribute: scalability, modifiability)
- **Negative:** อีเมล/points/analytics เกิดขึ้นแบบ eventual consistency ไม่ใช่ทันที (quality attribute: consistency)
- **Negative:** ต้องมี message broker เพิ่ม ต้องดูแลและมีค่าใช้จ่าย (quality attribute: cost, operability)
- **Negative:** debug และ trace ข้ามหลาย service ยากขึ้น ต้องมี correlation id และ observability (quality attribute: operability)
- **Risks and mitigations:**
  - event ซ้ำทำให้ให้ points สองครั้ง → consumer idempotent โดยใช้ order id
  - publish ล้มเหลวหลัง order สำเร็จ (dual-write) → retry ที่ผู้ publish, ประเมินตัวเลือก 3 (outbox) ถ้าพบ event หาย
  - broker เป็น single point of failure → deploy แบบ high availability หรือใช้ managed service
  - MailFast ล่มหรือ rate limit → retry + DLQ, แจ้งเตือนเมื่อ queue ค้าง
  - ข้อมูลส่วนบุคคลถูกส่งออกนอกระบบไป MailFast → ส่งเฉพาะข้อมูลที่จำเป็น และตรวจสอบนโยบายความเป็นส่วนตัว
  - vendor lock-in กับ MailFast → ซ่อนหลัง interface/adapter ใน email consumer

## Validation

- Fault-injection test: ปิด MailFast, `loyaltyservice` และ analytics ทีละตัว แล้วตรวจว่า checkout ยังสำเร็จและ event ถูกประมวลผลหลังระบบกลับมา
- Load test: เทียบ latency ของ checkout (p95) ก่อนและหลัง เพื่อยืนยันว่าไม่แย่ลง (เกณฑ์ตัวเลขต้องกำหนดโดยทีม ดู Open questions)
- Idempotency test: ส่ง `OrderPlaced` ซ้ำ ตรวจว่า points ถูกให้เพียงครั้งเดียว และอีเมลถูกส่งเพียงครั้งเดียว
- Metric: queue depth, consumer lag, อัตรา DLQ, อัตราอีเมลที่ส่งสำเร็จ; ทบทวน ADR หลังเพิ่ม consumer ตัวที่ 4 (เช่น SMS) เพื่อประเมินว่ารูปแบบนี้ยังเหมาะสมหรือไม่

## Open questions

ข้อมูลต่อไปนี้ไม่ได้ระบุใน requirement และไม่ได้เดา:

1. ระบบปัจจุบันมีโครงสร้างอย่างไร: service ใดเรียก `emailservice` อยู่ตอนนี้ และมี message broker หรือ data store สำหรับ order อยู่แล้วหรือไม่ (Not evidenced in the provided files)
2. เป้าหมายเชิงตัวเลข: latency ของ checkout ที่ยอมรับได้, จำนวน order ต่อวินาทีที่ต้องรองรับ, ความหน่วงที่ยอมรับได้ของอีเมล/points
3. ความต้องการด้านความน่าเชื่อถือ: ยอมให้ event หายได้หรือไม่ (ตัดสินว่าต้องใช้ outbox หรือไม่)
4. MailFast: รูปแบบ API (REST/SMTP), rate limit, SLA, ราคา, ข้อกำหนดด้าน data residency และ privacy
5. กฎของ loyalty points (อัตราคำนวณ, การยกเลิก/คืนเงิน, ที่เก็บข้อมูลของ `loyaltyservice`)
6. ข้อกำหนดของ analytics: event schema, ปลายทาง (ใครเป็นเจ้าของ), ต้องมี PII หรือไม่, ต้อง real-time หรือไม่
7. ทักษะและความพร้อมของทีมในการดูแล message broker และเทคโนโลยีที่ควรเลือก (เช่น managed vs self-hosted)
8. ผู้มีอำนาจตัดสินใจ (Deciders) และงบประมาณ
9. action ในอนาคต (SMS, fraud scoring) ต้องการผลลัพธ์แบบ synchronous หรือไม่ เช่น fraud scoring อาจต้องเกิด **ก่อน** ยืนยัน order ซึ่งจะไม่เหมาะกับ asynchronous pattern นี้
