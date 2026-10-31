**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** Nitchakran Chaipojjana / 6870405021
**agent ที่ใช้:** Claude Code
**โมเดล:** Claude Sonnet 5.5 (`claude-sonnet-5-5`)
**แพ็กเกจที่ใช้:** Claude Pro

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code, Sonnet 5.5 | `prompts/1-…` run A, evidence `protos/demo.proto` | `prompts/1-recover-architecture.md`, `system/online-boutique/protos/demo.proto` | ใช่ (ผลลัพธ์พูดถึง manifest และ `src/` เฉพาะในรายการสิ่งที่ยังไม่รู้ ไม่ได้อ้างว่าอ่าน) | 3 | |
| 2 | 1-B | Claude Code, Sonnet 5.5 | `prompts/1-…` run B, evidence `protos/demo.proto` + `kubernetes-manifests/` | `prompts/1-recover-architecture.md`, `demo.proto`, manifest 12 ไฟล์: `adservice`, `cartservice`, `checkoutservice`, `currencyservice`, `emailservice`, `frontend`, `kustomization`, `loadgenerator`, `paymentservice`, `productcatalogservice`, `recommendationservice`, `shippingservice` และ `kubernetes-manifests/README.md` | ใช่ | 5 | agent รายงานว่ารัน `ls` หนึ่งครั้งบนโฟลเดอร์ manifest ซึ่ง `AGENTS.md` ไม่ให้รันคำสั่ง terminal (แค่ list ชื่อไฟล์ในโฟลเดอร์หลักฐาน) ผลลัพธ์พบ `shoppingassistantservice` ที่ `frontend.yaml` อ้างถึงแต่ไม่มี manifest และ `redis-cart` อยู่ใน `cartservice.yaml` |
| 3 | 2-a | Claude Code, Sonnet 5.5 | `prompts/2-…` run a, `context: none`, `language: Thai` | `prompts/2-write-adr.md`, `docs/adr/template.md` ผลลัพธ์ระบุว่าไม่ได้อ่านโค้ดหรือเอกสารของระบบ | ใช่ | 3 | ผลลัพธ์เป็นภาษาไทย (ตรงกับ Run b) |
| 4 | 2-b | Claude Code, Sonnet 5.5 | `prompts/2-…` run b, context: `docs/scenario.md`, ADR 0001, `checkoutservice/main.go`, `language: Thai` | `prompts/2-write-adr.md`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-use-redis-for-cart-storage.md`, `src/checkoutservice/main.go` | ใช่ | 5 | รันซ้ำด้วย `language: Thai` เพราะรอบแรกได้ผลเป็นภาษาอังกฤษ ไม่ตรงกับ Run a ผลลัพธ์รอบแรกถูกเขียนทับ |
| 5 | 3 | Claude Code, Sonnet 5.5 | `prompts/3-…` adr: `docs/adr/0002-publish-order-placed-event-to-rabbitmq.md` | `prompts/3-review-adr.md`, ADR 0002, `docs/scenario.md` ผลลัพธ์ระบุว่าตรวจข้อความเรื่อง `main.go` ไม่ได้ | ใช่ | 5 | ผลลัพธ์เป็นภาษาอังกฤษ (ไม่ได้ใส่ `language: Thai`) |
| 6 | | | | | | | |
| 7 | | | | | | | |
