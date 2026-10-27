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
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย (ไม่ได้อ่าน scenario) ตัวเลขทั้งหมดถูกย้ายไปเป็น Open questions 12 ข้อ และ Validation ไม่มีเกณฑ์ผ่านเป็นตัวเลข | ใช้ครบ: p95 ≤ 500 ms (ตอนนี้ 300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / ล้ม 0.5 % / 50 emails/s, loyalty ภายใน 1 นาที และยังคำนวณต่อเอง เช่น backlog 9,000 อีเมล/นาที, ล้มราว 1 ครั้งทุก 4 วินาที |
| ทางเลือกที่พิจารณา (ระบุ) | 3 ทาง: sync direct calls, event pub/sub ผ่าน broker (**ไม่เลือกเทคโนโลยี**), orchestrator service | 4 ทาง: sync gRPC, async ใน process (goroutine), RabbitMQ, Kafka และตัด managed cloud broker ออกเพราะขัด constraint self-hosted |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | แต้ม: consumer idempotent ด้วย order ID; rate limit: ไม่รู้ค่า ใช้แค่ retry/backoff/DLQ ทั่วไป | แต้ม: `orderId` เป็น unique key + ack หลัง commit, queue loyalty แยกจาก email เพื่อไม่ให้ช้าเกิน 1 นาที; rate limit: consumer อีเมลคุมรวม ≤ 50/s + DLQ; **ยอมรับเอง**ว่าถ้า broker ล่มตอน publish event จะหาย (ขัดกับ exactly once) แต่ทิ้งไว้เป็น Open question |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่แต่งตัวเลข แต่เสนอ "managed cloud service" เป็นทางเลือก broker (ขัดกับ constraint ที่มันไม่รู้) และพูดถึง PDPA/data residency ที่ requirement ไม่ได้กล่าวถึง | ไม่มีตัวเลขแต่งขึ้นที่ไม่ติดป้าย ข้ออนุมานติด **[Inferred]** ไว้ (RabbitMQ 3 node + quorum queues, protobuf payload, Little's law ≈ 75 request ค้างพร้อมกัน) เลขบรรทัดใน `main.go` ที่อ้างตรวจแล้วถูกต้อง (252, 258, 263, 273, 233, 115, 122) |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

ADR ฉบับสุดท้าย: [`docs/adr/0002-use-rabbitmq-for-post-order-actions.md`](../docs/adr/0002-use-rabbitmq-for-post-order-actions.md)

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เปลี่ยนการตัดสินใจจาก "RabbitMQ publish ตรง" เป็น **RabbitMQ + transactional outbox** บน PostgreSQL ของ `checkoutservice` (เพิ่มเป็น Option 4 แยก) | ร่างของ AI ยอมรับเองว่าถ้า broker ล่มตอน publish แต้มจะหาย ซึ่งขัดกับ requirement "exactly once" แต่ปล่อยไว้เป็น Open question สถาปนิกต้องตัดสิน; และระบบไม่มีที่ใดเก็บ order ไว้ถาวร จึงตรวจย้อนไม่ได้เลยว่า order ไหนยังไม่ได้แต้ม ตาราง `orders` แก้ทั้งสองปัญหา |
| 2 | เพิ่ม driver ด้าน **security/privacy**: ห้ามใส่ `credit_card` ใน event และ analytics consumer ตัด email/ที่อยู่ออก | `PlaceOrderRequest` มีข้อมูลบัตร ร่างของ AI ไม่พูดถึง PII เลย ทั้งที่ event จะถูกส่งไปหลาย consumer รวมถึงทีม marketing |
| 3 | ใส่ตัวเลขผลกระทบของ backlog อีเมล (peak 10 นาที = 90,000 ฉบับ, เคลียร์ ≥ 30 นาที) และเปลี่ยน alert ของ loyalty จาก "ใกล้ 60 s" เป็น **30 s** | กรรมการ (โดยเฉพาะ PO) ต้องเห็นผลกระทบเป็นตัวเลขเพื่อตัดสินว่ารับได้ไหม; alert ตอนใกล้ 60 s ช้าเกินไปที่จะแก้ทันก่อนผิด SLO |
| 4 | เลื่อน reliability/correctness ขึ้นมาก่อน performance ใน Decision drivers, ระบุ Deciders และตัดข้ออนุมานที่ไม่จำเป็นออก (3-node quorum, Little's law) | exactly once เป็นข้อบังคับที่ผิดไม่ได้ ส่วน latency ยังมีงบเหลือราว 200 ms; ข้ออนุมานที่ไม่ได้ตรวจสอบทำให้ ADR ปกป้องยากในที่ประชุม |
