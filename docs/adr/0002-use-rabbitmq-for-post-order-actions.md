# ADR-0002: ส่งงานหลังสั่งซื้อแบบ asynchronous ผ่าน event `OrderPlaced` บน RabbitMQ พร้อม transactional outbox

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** Piyapanchanok Meeboonsalang (สถาปนิก, ผู้เขียน ADR) · รอการอนุมัติจากคณะกรรมการรีวิวสถาปัตยกรรมก่อนเทศกาล 11.11

## Context and problem

ข้อกำหนดใหม่ R-12 (`docs/scenario.md`) กำหนดว่าหลังสั่งซื้อสำเร็จ ระบบต้อง (1) ส่งอีเมลยืนยันผ่าน **MailFast** แทน logic จำลองของ `emailservice` (2) ให้ loyalty points ผ่าน `loyaltyservice` ที่จะสร้างใหม่ และ (3) บันทึก analytics event "order placed" ให้ทีม marketing ปีหน้าจะมีงานหลังสั่งซื้อเพิ่มอีก (SMS, fraud scoring, อัปเดต recommendations)

**สภาพปัจจุบัน** (`system/online-boutique/src/checkoutservice/main.go`):

- `PlaceOrder` เรียก service อื่นแบบ synchronous gRPC ตามลำดับ: `chargeCard` (บรรทัด 252) → `shipOrder` (258) → `emptyUserCart` (263) → `sendOrderConfirmation` (273)
- ถ้าส่งอีเมลล้มเหลว จะแค่ `log.Warnf` แล้วคืนผลว่าสำเร็จ (บรรทัด 273–277)
- `sendOrderConfirmation` ใช้ `ctx` ของ request ต่อโดยไม่ตั้ง timeout ของตัวเอง (บรรทัด 379–384) ปลายทางช้าเท่าไร `PlaceOrder` ก็ช้าเท่านั้น
- `checkoutservice` ผูกกับ `emailservice` ตั้งแต่ตอน start (`EMAIL_SERVICE_ADDR` บรรทัด 115, connection บรรทัด 122) ถ้าเพิ่ม action ใหม่แบบเดิม ทุก action ต้องเพิ่ม field, env และ connection ใน struct `checkoutService` (บรรทัด 66–86)
- **`checkoutservice` ไม่มี datastore และไม่มีที่ใดในระบบเก็บ order ไว้ถาวร** order มีอยู่แค่ใน response และอีเมล จึงไม่มีข้อมูลให้ตรวจย้อน (reconcile) ได้ว่า order ไหนได้แต้มแล้วหรือยัง
- order ID สร้างจาก `uuid.NewUUID()` (บรรทัด 233) ใช้เป็น idempotency key ได้

**ตัวเลขที่บังคับการตัดสินใจ** (`docs/scenario.md`, "Quality goals and constraints"):

| ประเด็น | ข้อเท็จจริง | ผลต่อการออกแบบ |
|---|---|---|
| Checkout latency | p95 ของ `PlaceOrder` ≤ 500 ms (ตอนนี้ราว 300 ms) | เหลืองบราว 200 ms สำหรับงานที่จะเพิ่ม |
| MailFast latency | p95 1.5 s, p99 8 s | เรียก MailFast ใน `PlaceOrder` โดยตรงจะเกิน 500 ms แน่นอน |
| Peak เทียบกับ rate limit | 200 orders/s แต่ MailFast รับได้ 50 emails/s | ช่วง 11.11 อีเมลค้างเพิ่ม 150 ฉบับ/วินาที = 9,000 ฉบับต่อทุกนาทีของ peak จึง**ต้องมี buffer ที่ทนทาน**ไม่ว่าจะเลือกทางไหน |
| ความล้มเหลวของ MailFast | ล้มเหลวราว 0.5 % ของ call | ที่ 50 emails/s คือล้มราว 1 ครั้งทุก 4 วินาที ต้องมี retry |
| Loyalty points | ต้องได้ภายใน 1 นาที และ exactly once | ต้องส่งต่อแบบทนทาน **ไม่หายแม้ broker ล่ม** และกันซ้ำด้วย order ID |
| Availability | order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม | ต้องแยกงานหลังสั่งซื้อออกจาก path ของ checkout |
| Team / Ops | dev 4 คน เก่ง Go และ gRPC, 2 คนเคยใช้ RabbitMQ; ทุกอย่างรันบน Kubernetes; บริษัทชอบ open-source แบบ self-hosted | ตัดตัวเลือกที่เป็น managed service แบบ cloud-only ออก |

**ปัญหา:** จะเชื่อม `checkoutservice` เข้ากับงานหลังสั่งซื้อ (ตอนนี้ 3 งาน ปีหน้ามีเพิ่ม) อย่างไร โดยไม่ทำให้ checkout ช้าลงหรือล้มเหลว และยังรับประกันว่า loyalty points ได้ครั้งเดียวภายใน 1 นาที

## Decision drivers

1. **Availability:** order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม
2. **Reliability / correctness:** loyalty points ต้องได้ภายใน 1 นาทีแบบ exactly once; อีเมลต้องไม่หาย แม้ MailFast จะล้ม 0.5 % และจำกัดไว้ที่ 50 emails/s
3. **Performance:** p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s
4. **Modifiability:** เพิ่มงานหลังสั่งซื้อใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ `checkoutservice` ทุกครั้ง
5. **Operability, cost และความเหมาะกับทีม:** รันบน Kubernetes, เป็น open-source self-hosted และทีม 4 คนดูแลไหว
6. **Security / privacy:** ข้อมูลบัตรเครดิตและ PII ต้องไม่รั่วไปยัง consumer ที่ไม่จำเป็นต้องเห็น

## Considered options

1. **Synchronous gRPC (ขยายแบบเดิม):** `PlaceOrder` เรียก MailFast, `loyaltyservice` และ analytics ต่อจาก `shipOrder` แบบ synchronous
2. **Async ภายใน process:** `PlaceOrder` ตอบกลับทันที แล้วส่งงานให้ goroutine และ in-memory queue ภายใน `checkoutservice`
3. **RabbitMQ แบบ self-hosted, publish ตรงจาก `PlaceOrder`:** `checkoutservice` publish event `OrderPlaced` ไปที่ exchange; แต่ละงานมี queue และ consumer ของตัวเอง
4. **RabbitMQ แบบ self-hosted + transactional outbox:** เหมือนข้อ 3 แต่ `checkoutservice` บันทึก order และ event ลงฐานข้อมูลของตัวเองใน transaction เดียว แล้วมี relay ส่ง event ไป RabbitMQ
5. **Event log แบบ self-hosted (Apache Kafka):** publish `OrderPlaced` ลง topic; แต่ละงานเป็น consumer group แยกกัน

ไม่พิจารณา managed broker แบบ cloud-only (เช่น Google Cloud Pub/Sub) เพราะขัดกับ constraint เรื่อง open-source self-hosted

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC | ไม่มี component ใหม่; ทีมถนัด gRPC; รู้ผลทันที | MailFast p95 1.5 s ทำให้ p95 ของ `PlaceOrder` เกิน 500 ms; ช่วง peak มี ≥ 150 calls/s ที่เกิน rate limit; ถ้าตั้ง timeout สั้น อีเมลจะหายแทน; ทุก action ใหม่ต้องแก้ `checkoutservice` |
| 2. Async ใน process | latency ไม่ขึ้นกับ MailFast; ไม่มี infra ใหม่ | งานค้างใน memory ของ pod (ช่วง peak เพิ่ม 9,000 อีเมลต่อนาที) และ**หายเมื่อ pod restart หรือ deploy** ทำให้ loyalty points หาย; ต้องคุม rate limit 50/s ข้ามหลาย replica เอง |
| 3. RabbitMQ, publish ตรง | checkout ไม่ขึ้นกับ MailFast; durable queue รับ backlog ได้; consumer อีเมลคุมอัตรา ≤ 50/s ได้; เพิ่ม action = เพิ่ม queue; 2 คนในทีมเคยใช้ | broker อยู่บน critical path ของ checkout; **ถ้า broker ล่มตอน publish event จะหาย** (order สำเร็จแต่ไม่ได้แต้ม ขัดกับ exactly once) และไม่มีข้อมูล order ให้ตรวจย้อน |
| 4. RabbitMQ + outbox | ได้ข้อดีของข้อ 3 ทั้งหมด; event ไม่หายแม้ RabbitMQ ล่ม (แค่ล่าช้า); broker ออกจาก critical path ของ checkout; ได้ตาราง `orders` ไว้ reconcile ว่าทุก order ได้แต้มครบ | เพิ่ม component แบบ stateful อีกหนึ่งตัว (PostgreSQL ของ `checkoutservice`); checkout ต้องเขียน DB หนึ่งครั้งต่อ order (เพิ่ม latency, ต้องวัด); มี relay ที่ต้องเขียนและดูแล |
| 5. Kafka | throughput สูง; replay event ย้อนหลังได้ | ดูแลยากกว่า (cluster, partition, retention); ทีมไม่มีประสบการณ์ Kafka ตาม scenario; retry/delay/dead-letter ต้องเขียนเอง; ยังต้องกันซ้ำที่ consumer และยังมีปัญหา publish หายแบบข้อ 3 ถ้าไม่ทำ outbox |

## Decision

เลือก **Option 4: RabbitMQ + transactional outbox** เพราะเป็นตัวเลือกเดียวที่ตอบ driver 1–4 ได้ครบ รวมถึง "exactly once" ของ loyalty points ในกรณีที่ broker ล่ม ซึ่ง Option 3 ทำไม่ได้ และยังเข้ากับทีม (2 คนเคยใช้ RabbitMQ) กับ constraint เรื่อง open-source self-hosted

รายละเอียด:

1. **Publisher (`checkoutservice`):**
   - หลัง `chargeCard` และ `shipOrder` สำเร็จ ให้เขียน row ใน `orders` และ row ใน `outbox` (event `OrderPlaced`) ลง **PostgreSQL แบบ self-hosted (`checkout-db`)** ภายใน transaction เดียว แทนการเรียก `sendOrderConfirmation` (บรรทัด 273–277)
   - ถ้าเขียน DB ไม่สำเร็จ order ยังคืนผลสำเร็จ (ตาม driver 1) และ fallback เป็นการ publish ตรงไป RabbitMQ แบบ best-effort พร้อม metric/alert ดังนั้น event จะหายก็ต่อเมื่อ **ทั้ง** `checkout-db` และ RabbitMQ ล่มพร้อมกัน
   - ใช้ `orderId` (บรรทัด 233) เป็น message ID และ idempotency key
   - ถอด `EMAIL_SERVICE_ADDR` และ connection ไปยัง `emailservice` ออกจาก `checkoutservice` (บรรทัด 81–82, 115, 122)
2. **Outbox relay:** goroutine ใน `checkoutservice` อ่าน row ที่ยังไม่ส่ง (`SELECT … FOR UPDATE SKIP LOCKED` เพื่อให้หลาย replica ทำงานพร้อมกันได้) แล้ว publish แบบ persistent พร้อม publisher confirms ไปยัง durable topic exchange `orders` จากนั้นจึง mark ว่าส่งแล้ว ถ้า RabbitMQ ล่ม event จะรอใน outbox แล้วส่งเมื่อกลับมา (การส่งซ้ำได้ เพราะ consumer เป็น idempotent)
3. **Payload ของ `OrderPlaced`:** `order_id`, `user_id`, `email`, `items`, ยอดรวมและสกุลเงิน, ที่อยู่จัดส่ง **ห้ามมีข้อมูล `credit_card`** (มีอยู่ใน `PlaceOrderRequest`) ใช้ protobuf เพราะทีมใช้อยู่แล้ว
4. **Consumers** (แต่ละตัวมี queue ของตัวเองที่ bind กับ exchange):
   - `order.email` → `emailservice` (เปลี่ยนจาก mock เป็นเรียก MailFast): คุมอัตรารวมทั้งระบบให้ ≤ 50 emails/s ด้วย rate limiter ตัวเดียว, retry แบบ backoff และส่งเข้า dead-letter queue เมื่อ retry ครบ
   - `order.loyalty` → `loyaltyservice` ใหม่ (Go): ใช้ `orderId` เป็น unique key ในที่เก็บข้อมูลของ loyalty และ ack หลัง commit เท่านั้น outbox (at-least-once) + consumer idempotent = ผลลัพธ์แบบ exactly once
   - `order.analytics` → consumer ที่ส่ง event ไปยังระบบ analytics ของ marketing โดย**ตัด `email` และที่อยู่ออก**ก่อนส่ง
5. **Action ใหม่ของปีหน้า:** เพิ่ม queue และ consumer ใหม่ bind กับ exchange เดิม โดยไม่ต้องแก้ `checkoutservice` ข้อยกเว้นคือถ้า fraud scoring ต้อง "บล็อก" order ก่อนตัดเงิน ต้องเป็น synchronous และทำ ADR แยก

ทำไมไม่เลือกตัวอื่น: Option 1 ผิดเป้า p95 500 ms และ rate limit ตั้งแต่แรก; Option 2 ทำให้แต้มหายเมื่อ pod restart; Option 3 ทำให้แต้มหายเมื่อ broker ล่มตอน publish และไม่มีข้อมูลให้ตรวจย้อน; Option 5 ต้นทุนการดูแลและการเรียนรู้สูงกว่าสำหรับทีม 4 คน ทั้งที่ยังไม่มี requirement เรื่อง replay ที่ยืนยันแล้ว

## Consequences

- **Positive:** `PlaceOrder` ไม่ต้องรอ MailFast, loyalty, analytics หรือแม้แต่ RabbitMQ เหลือแค่ต้นทุนการเขียน DB หนึ่งครั้ง (quality attribute: performance)
- **Positive:** ถ้า email, loyalty, analytics หรือ RabbitMQ ล่ม order ก็ยังสำเร็จ และ event จะรอใน outbox/queue จนกว่าจะกลับมา (quality attribute: availability)
- **Positive:** loyalty points ได้ครั้งเดียวแม้ broker ล่ม และตาราง `orders` ใช้ reconcile กับข้อมูลของ `loyaltyservice` ได้ (quality attribute: reliability / correctness)
- **Positive:** queue รับส่วนต่างระหว่าง 200 orders/s กับ 50 emails/s และ retry ความล้มเหลว 0.5 % ของ MailFast โดยอีเมลไม่หาย (quality attribute: reliability)
- **Positive:** เพิ่มงานหลังสั่งซื้อทำได้ด้วยการเพิ่ม consumer (quality attribute: modifiability)
- **Positive:** ข้อมูลบัตรไม่อยู่ใน event และ analytics ไม่ได้รับ PII (quality attribute: security / privacy)
- **Negative:** มี component แบบ stateful ใหม่ **2 ตัว** (RabbitMQ cluster และ `checkout-db`) ที่ต้อง upgrade, backup และ monitor (quality attribute: operability, cost)
- **Negative:** อีเมลกลายเป็น eventually consistent: backlog เพิ่ม 9,000 ฉบับต่อทุกนาทีของ peak เช่น peak 10 นาทีจะค้าง 90,000 ฉบับ ซึ่งต้องใช้อย่างน้อย 30 นาทีในการเคลียร์แม้ไม่มี order ใหม่เข้ามาเลย ข้อจำกัดนี้มาจาก rate limit ของ MailFast ไม่ว่าเลือก option ไหนก็เลี่ยงไม่ได้ (quality attribute: usability / timeliness)
- **Negative:** at-least-once delivery บังคับให้ทุก consumer ต้อง idempotent และการ debug flow แบบ async ยากขึ้น (quality attribute: testability / observability)
- **Risks and mitigations:**
  - *การเขียน `checkout-db` เพิ่ม latency:* ต้องวัดว่าอยู่ในงบราว 200 ms ที่ 200 orders/s (ยังไม่มีตัวเลข ดู Open questions)
  - *`checkout-db` ล่ม:* fallback เป็น publish ตรงแบบ best-effort + alert; ใช้ PostgreSQL แบบ primary/replica
  - *outbox โตเร็วตอน RabbitMQ ล่ม:* alert เมื่อ row ที่ยังไม่ส่งเกินเกณฑ์ และลบ row ที่ส่งแล้วเป็นระยะ
  - *ได้ loyalty points ซ้ำ* (relay ส่งซ้ำหรือ redelivery): ใช้ `orderId` เป็น unique key และ ack หลัง commit
  - *ส่งอีเมลซ้ำ* (MailFast timeout แต่ส่งไปแล้ว): ใช้ idempotency key ถ้า MailFast รองรับ ดู Open questions
  - *loyalty เกิน 1 นาทีเพราะ backlog:* queue ของ loyalty แยกจาก email จึงไม่โดนอีเมลที่ค้างถ่วง; scale `loyaltyservice` ให้รับได้ ≥ 200 msg/s และ alert เมื่อเวลาจาก order ถึงได้แต้มเกิน **30 s** (ครึ่งหนึ่งของงบ เพื่อให้มีเวลาแก้)
  - *poison message:* ส่งเข้า dead-letter queue และ alert
  - *trace ขาดช่วงข้าม broker:* `checkoutservice` ใช้ OpenTelemetry อยู่แล้ว (`initTracing` บรรทัด 154–178) ให้ใส่ trace context ไว้ใน message header

## Validation

- **Load test** ที่ 200 orders/s (ใช้ Locust `loadgenerator` แบบเดียวกับ ADR-0001) กับ MailFast stub ที่จำลอง p95 1.5 s, p99 8 s, ล้ม 0.5 % และจำกัด 50 req/s แล้วตรวจว่า:
  - p95 ของ `PlaceOrder` ≤ 500 ms
  - เวลาจาก order ถึงได้แต้ม ≤ 60 s สำหรับ**ทุก** order และจำนวน record แต้มเท่ากับจำนวน row ใน `orders` (ไม่มีแต้มซ้ำหรือหาย)
  - อัตราเรียก MailFast ไม่เกิน 50/s และทุกอีเมลถูกส่งสำเร็จหรืออยู่ใน dead-letter queue
- **Fault injection:** ปิด `emailservice`, `loyaltyservice`, consumer ของ analytics, MailFast stub และ **RabbitMQ ทั้ง cluster** ทีละตัวระหว่าง load test order ต้องสำเร็จ 100 % และเมื่อเปิดกลับมา ทุก order ต้องได้แต้มครบ (reconcile `orders` กับ `loyaltyservice`)
- **Review:** ทบทวน ADR นี้หลังเทศกาล 11.11 ครั้งแรก และก่อนเพิ่ม action ใหม่ของปีหน้า (โดยเฉพาะ fraud scoring)

## Open questions

1. ช่วง peak 11.11 ยาวนานเท่าไร และอีเมลยืนยันช้าได้มากที่สุดเท่าไร? (peak 10 นาที = อีเมลค้าง 90,000 ฉบับ ถ้ารับไม่ได้ต้องเจรจาเพิ่ม rate limit กับ MailFast ซึ่งกระทบ cost)
2. MailFast: เมื่อเกิน 50 emails/s จะปฏิเสธหรือ queue ไว้เอง? รองรับ idempotency key หรือไม่? ใช้ protocol อะไร? ค่าใช้จ่ายเท่าไร?
3. latency ของการเขียน `checkout-db` และของ RabbitMQ ใน cluster นี้ และงบทรัพยากรของทั้งสองตัว: ต้องวัดและประเมิน
4. Analytics: ปลายทางคืออะไร? event ต้องมี field อะไร? ยอมให้หายหรือซ้ำได้ไหม? ต้อง replay ย้อนหลังไหม (ถ้าต้อง ควรพิจารณา Kafka ใหม่)?
5. `loyaltyservice`: ใช้ datastore อะไร กติกาการคิดแต้มเป็นอย่างไร และรับได้กี่ msg/s ต่อ replica?
6. จะแก้ `emailservice` เดิมให้เป็น consumer ที่เรียก MailFast หรือสร้าง service ใหม่?
7. Fraud scoring ของปีหน้าต้องบล็อก order ก่อนตัดเงินหรือไม่? ถ้าใช่ ต้องทำ ADR แยก
8. ใครเป็น on-call ของ RabbitMQ และ `checkout-db`?
