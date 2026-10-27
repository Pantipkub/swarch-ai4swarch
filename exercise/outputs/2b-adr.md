> Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`) · run: b

# ADR-0002: ส่งงานหลังสั่งซื้อแบบ asynchronous ผ่าน event `OrderPlaced` บน RabbitMQ

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ทีมพัฒนา Online Boutique 4 คน (ยังไม่ระบุผู้อนุมัติจริง ดู Open questions)

## Context and problem

ข้อกำหนดใหม่ R-12 (`docs/scenario.md`, "The new requirement (R-12)") กำหนดว่าหลังสั่งซื้อสำเร็จ ระบบต้อง (1) ส่งอีเมลยืนยันผ่าน **MailFast** แทน logic จำลองของ `emailservice` (2) ให้ loyalty points ผ่าน `loyaltyservice` ที่จะสร้างใหม่ และ (3) บันทึก analytics event "order placed" ให้ทีม marketing นอกจากนี้ปีหน้าจะมีงานหลังสั่งซื้อเพิ่มอีก (SMS, fraud scoring, อัปเดต recommendations)

**สภาพปัจจุบัน** (`system/online-boutique/src/checkoutservice/main.go`):

- `PlaceOrder` เรียก service อื่นแบบ synchronous gRPC ตามลำดับ: `chargeCard` (บรรทัด 252) → `shipOrder` (258) → `emptyUserCart` (263) → `sendOrderConfirmation` (273)
- ถ้าส่งอีเมลล้มเหลว จะแค่ `log.Warnf` แล้วคืนผลว่าสำเร็จ (บรรทัด 273–277) ซึ่งตรงกับเป้า availability อยู่แล้ว
- `sendOrderConfirmation` ส่ง `ctx` ของ request ต่อไปตรง ๆ โดยไม่ตั้ง `context.WithTimeout` ของตัวเอง (บรรทัด 379–384) ดังนั้นปลายทางช้าเท่าไร `PlaceOrder` ก็ช้าเท่านั้น
- `checkoutservice` ผูกกับ `emailservice` ตั้งแต่ตอน start: `EMAIL_SERVICE_ADDR` เป็น env ที่บังคับต้องมี (`mustMapEnv`, บรรทัด 115) และเปิด gRPC connection ไว้ (บรรทัด 122) ถ้าทำ action ใหม่ด้วยวิธีเดิม ทุก action ต้องเพิ่ม field, env และ connection ใน struct `checkoutService` (บรรทัด 66–86)
- `checkoutservice` ไม่มี datastore ของตัวเอง (ใน `main.go` มีแค่ gRPC client ไปยัง 6 service)
- order ID สร้างจาก `uuid.NewUUID()` (บรรทัด 233) จึงใช้เป็น key กันการทำซ้ำได้

**ตัวเลขที่บังคับการตัดสินใจ** (`docs/scenario.md`, "Quality goals and constraints"):

| ประเด็น | ข้อเท็จจริง | ผลต่อการออกแบบ |
|---|---|---|
| Checkout latency | p95 ของ `PlaceOrder` ≤ 500 ms (ตอนนี้ราว 300 ms) | เหลืองบราว 200 ms สำหรับงานที่จะเพิ่ม |
| MailFast latency | p95 1.5 s, p99 8 s | ถ้าเรียก MailFast ใน `PlaceOrder` โดยตรง จะเกิน 500 ms แน่นอน (ประมาณคร่าว ๆ 300 ms + 1.5 s) |
| Peak เทียบกับ rate limit | 200 orders/s แต่ MailFast รับได้ 50 emails/s | ช่วง 11.11 อีเมลค้างเพิ่มขึ้น 150 ฉบับ/วินาที หรือ 9,000 ฉบับต่อทุกนาทีของ peak จึง**ต้องมี buffer ที่ทนทาน**ไม่ว่าจะเลือกทางไหน |
| ความล้มเหลวของ MailFast | ล้มเหลวราว 0.5 % ของ call | ที่ 50 emails/s คือล้มราว 1 ครั้งทุก 4 วินาที จึงต้องมี retry |
| Loyalty points | ต้องได้ภายใน 1 นาที และ exactly once | ต้องส่งต่อแบบทนทาน และกันซ้ำด้วย order ID |
| Availability | order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม | ต้องแยกงานหลังสั่งซื้อออกจาก path ของ checkout |
| Team / Ops | dev 4 คน เก่ง Go และ gRPC, 2 คนเคยใช้ RabbitMQ; ทุกอย่างรันบน Kubernetes; บริษัทชอบ open-source แบบ self-hosted | ตัดตัวเลือกที่เป็น managed service แบบ cloud-only ออก |

**ปัญหา:** จะเชื่อม `checkoutservice` เข้ากับงานหลังสั่งซื้อ (ตอนนี้ 3 งาน ปีหน้ามีเพิ่ม) อย่างไร โดยไม่ทำให้ checkout ช้าลงหรือล้มเหลว และยังรับประกันเรื่อง loyalty points ได้

## Decision drivers

1. **Availability:** order ต้องไม่ล้มเพราะ email, loyalty หรือ analytics ล่ม
2. **Performance:** p95 ของ `PlaceOrder` ≤ 500 ms ที่ 200 orders/s
3. **Reliability / correctness:** loyalty points ต้องได้ภายใน 1 นาทีแบบ exactly once; อีเมลต้องไม่หาย แม้ MailFast จะล้ม 0.5 % และจำกัดไว้ที่ 50 emails/s
4. **Modifiability:** เพิ่มงานหลังสั่งซื้อใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้ `checkoutservice` ทุกครั้ง
5. **Operability, cost และความเหมาะกับทีม:** รันบน Kubernetes, เป็น open-source self-hosted และทีม 4 คนดูแลไหว

## Considered options

1. **Synchronous gRPC (ขยายแบบเดิม):** `PlaceOrder` เรียก MailFast, `loyaltyservice` และ analytics ต่อจาก `shipOrder` แบบ synchronous
2. **Async ภายใน process:** `PlaceOrder` ตอบกลับทันที แล้วส่งงานให้ goroutine และ in-memory queue ภายใน `checkoutservice` ไปเรียก MailFast, loyalty และ analytics
3. **Message broker แบบ self-hosted (RabbitMQ):** `checkoutservice` publish event `OrderPlaced` ครั้งเดียวไปที่ exchange; แต่ละงาน (email, loyalty, analytics) มี queue และ consumer ของตัวเอง
4. **Event log แบบ self-hosted (Apache Kafka):** publish `OrderPlaced` ลง topic; แต่ละงานเป็น consumer group แยกกัน

ไม่พิจารณา managed broker แบบ cloud-only (เช่น Google Cloud Pub/Sub) ต่อ เพราะขัดกับ constraint เรื่อง open-source self-hosted

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC | ไม่มี component ใหม่; ทีมถนัด gRPC; รู้ผลทันทีว่าสำเร็จหรือไม่ | MailFast p95 1.5 s ทำให้ p95 ของ `PlaceOrder` เกิน 500 ms; ช่วง peak มีอย่างน้อย 150 calls/s ที่เกิน rate limit; ถ้าตั้ง timeout สั้นเพื่อรักษา latency อีเมลจะหายแทน (เพราะความล้มเหลวถูกแค่ log ไว้ บรรทัด 274); ทุก action ใหม่ต้องแก้ `checkoutservice` และทำให้ช้าลงอีก |
| 2. Async ใน process | latency ของ checkout ไม่ขึ้นกับ MailFast; ไม่มี infra ใหม่; เขียนด้วย Go ได้ทันที | งานค้างอยู่ใน memory ของ pod (ช่วง peak เพิ่มขึ้น 9,000 อีเมลต่อนาที) และ**หายเมื่อ pod restart หรือ deploy** ทำให้ loyalty points อาจไม่ถูกให้ (ผิดข้อ "ภายใน 1 นาที, exactly once"); ต้องคุม rate limit 50/s ร่วมกันข้ามหลาย replica เอง; `checkoutservice` ยังต้องรู้จักทุกปลายทาง |
| 3. RabbitMQ | checkout แค่ publish แล้วตอบกลับ จึงไม่ขึ้นกับ MailFast; durable queue รับ backlog ช่วง peak ได้; consumer อีเมลคุมอัตราให้ ≤ 50/s และทำ retry/dead-letter ได้; เพิ่ม action ใหม่ = เพิ่ม queue และ consumer โดยไม่ต้องแตะ `checkoutservice`; เป็น open-source self-hosted บน Kubernetes; มี 2 คนในทีมเคยใช้ | มี component แบบ stateful เพิ่มที่ต้องดูแล; delivery เป็นแบบ at-least-once จึงต้องทำ consumer ให้ idempotent; message ที่ ack แล้วจะถูกลบ ย้อนกลับมาเล่นซ้ำ (replay) ไม่ได้; broker อยู่บน path ของ checkout ตอน publish |
| 4. Kafka | throughput สูง; เก็บ event ไว้อ่านย้อนหลังได้ (เหมาะกับ analytics และ consumer ใหม่ปีหน้าที่อยากอ่านข้อมูลเก่า); เป็น open-source self-hosted ได้ | ดูแลยากกว่า (cluster, partition, retention); scenario ไม่ได้บอกว่าทีมมีประสบการณ์ Kafka; retry รายข้อความ, delay และ dead-letter ต้องเขียนเองใน consumer; exactly once ของ loyalty ยังต้องกันซ้ำที่ consumer เหมือนเดิม; throughput ที่เพิ่มมาเกินความจำเป็นที่ 200 orders/s [Inferred] |

## Decision

เลือก **Option 3: RabbitMQ** เพราะเป็นตัวเลือกเดียวที่ตอบ driver 1–4 ได้ครบ และยังเข้ากับทีม (2 คนเคยใช้) กับ constraint เรื่อง open-source self-hosted (driver 5)

รายละเอียด:

1. **Publisher (`checkoutservice`):**
   - แทนที่การเรียก `sendOrderConfirmation` (บรรทัด 273–277) ด้วยการ publish event `OrderPlaced` หนึ่งครั้งไปยัง durable topic exchange (เช่น `orders`) ที่ตำแหน่งเดิม คือหลัง payment และ shipping สำเร็จแล้ว เพื่อให้เกิด event เฉพาะ order ที่สำเร็จจริง
   - ใช้ `orderId` (บรรทัด 233) เป็น message ID และ idempotency key
   - publish แบบ persistent พร้อม publisher confirms และ timeout สั้น ถ้า publish ไม่สำเร็จให้ log ไว้แล้วยังคืนผล order สำเร็จ (เหมือนที่ทำกับอีเมลตอนนี้) ดู Risks and mitigations
   - ถอด `EMAIL_SERVICE_ADDR` และ connection ไปยัง `emailservice` ออกจาก `checkoutservice` (บรรทัด 81–82, 115, 122)
   - ใช้ protobuf เป็น payload เพราะทีมใช้ gRPC/protobuf อยู่แล้ว [Inferred]
2. **Consumers** (แต่ละตัวมี queue ของตัวเองที่ bind กับ exchange):
   - `order.email` → `emailservice` (เปลี่ยน logic จาก mock เป็นเรียก MailFast): คุมอัตรารวมทั้งระบบให้ ≤ 50 emails/s, retry แบบ backoff และส่งเข้า dead-letter queue เมื่อ retry ครบ ถ้าจะส่งได้เต็ม 50 emails/s ต้องมี request ค้างพร้อมกันราว 50 × 1.5 s ≈ 75 ฉบับ (Little's law โดยใช้ p95 เป็นค่าประมาณแบบเผื่อไว้) [Inferred]
   - `order.loyalty` → `loyaltyservice` ใหม่ (Go): ใช้ `orderId` เป็น unique key ในที่เก็บข้อมูลของ loyalty และ ack หลัง commit เท่านั้น เมื่อรวม at-least-once delivery กับ consumer ที่ idempotent ผลลัพธ์จะเป็น exactly once
   - `order.analytics` → consumer ที่ส่ง event ไปยังระบบ analytics ของ marketing (ยังไม่รู้ว่าปลายทางคืออะไร ดู Open questions)
3. **Action ใหม่ของปีหน้า:** เพิ่ม queue และ consumer ใหม่ bind กับ exchange เดิม โดยไม่ต้องแก้ `checkoutservice` ข้อยกเว้นคือถ้า fraud scoring ต้อง "บล็อก" order ก่อนตัดเงิน จะไม่เข้ากับรูปแบบนี้ (ดู Open questions)

ทำไมไม่เลือกตัวอื่น: Option 1 ผิดทั้งเป้า p95 500 ms และ rate limit ตั้งแต่แรก; Option 2 ทำให้ loyalty points หายได้เมื่อ pod restart; Option 4 ทำได้ตามโจทย์ แต่ต้นทุนการดูแลและการเรียนรู้สูงกว่าสำหรับทีม 4 คน ทั้งที่ยังไม่มี requirement เรื่อง replay ที่ยืนยันแล้ว

## Consequences

- **Positive:** `PlaceOrder` ไม่ต้องรอ MailFast (p95 1.5 s, p99 8 s), loyalty หรือ analytics อีก เหลือแค่ต้นทุนการ publish หนึ่งครั้ง (quality attribute: performance)
- **Positive:** ถ้า email, loyalty หรือ analytics ล่ม order ก็ยังสำเร็จ และ message จะรอใน queue จนกว่า consumer จะกลับมา (quality attribute: availability)
- **Positive:** queue รับส่วนต่างระหว่าง 200 orders/s กับ 50 emails/s ช่วง peak และ retry ความล้มเหลว 0.5 % ของ MailFast โดยอีเมลไม่หาย (quality attribute: reliability)
- **Positive:** เพิ่มงานหลังสั่งซื้อทำได้ด้วยการเพิ่ม consumer; `checkoutservice` ไม่ต้องรู้จักปลายทางใหม่ (quality attribute: modifiability)
- **Positive:** ใช้ทักษะที่ทีมมีอยู่แล้ว (Go, gRPC, RabbitMQ 2 คน) และรันบน Kubernetes เดิม (quality attribute: cost / operability)
- **Negative:** มี component แบบ stateful ใหม่ที่ต้องดูแล (upgrade, monitoring, storage) (quality attribute: operability, cost)
- **Negative:** อีเมลกลายเป็น eventually consistent: ช่วง 11.11 backlog เพิ่มขึ้น 9,000 ฉบับต่อทุกนาทีของ peak ลูกค้าอาจได้อีเมลช้าหลายนาที ข้อจำกัดนี้มาจาก rate limit ของ MailFast ซึ่งไม่ว่าเลือก option ไหนก็เลี่ยงไม่ได้ (quality attribute: usability / timeliness)
- **Negative:** at-least-once delivery บังคับให้ทุก consumer ต้อง idempotent และการ debug flow แบบ async ยากขึ้น (quality attribute: testability / observability)
- **Risks and mitigations:**
  - *RabbitMQ ล่มตอน publish:* order ยังต้องสำเร็จ (driver 1) แต่ event จะหาย ทำให้ไม่ได้ loyalty points (ขัดกับ exactly once) นี่คือ trade-off หลักของการตัดสินใจนี้ Mitigation: RabbitMQ แบบ cluster 3 node พร้อม quorum queues [Inferred]; timeout ของ publish สั้น และมี metric/alert เมื่อ publish ล้ม; ถ้ายอมให้หายไม่ได้ ให้เพิ่ม transactional outbox ซึ่งต้องเพิ่ม datastore ให้ `checkoutservice` (ตอนนี้ไม่มี) ดู Open questions
  - *การ publish เพิ่ม latency:* ต้องวัดจริงว่าอยู่ในงบราว 200 ms (latency ของ RabbitMQ ใน cluster นี้: Not evidenced in the provided files)
  - *ได้ loyalty points ซ้ำ* (redelivery เมื่อ consumer crash ก่อน ack): ใช้ `orderId` เป็น unique key และ ack หลัง commit
  - *ส่งอีเมลซ้ำ* (MailFast timeout แต่จริง ๆ ส่งไปแล้ว แล้วเรา retry): ใช้ idempotency key ถ้า MailFast รองรับ ดู Open questions
  - *loyalty เกิน 1 นาทีเพราะ backlog:* queue ของ loyalty แยกจาก email จึงไม่โดนอีเมลที่ค้างถ่วง; scale `loyaltyservice` ให้รับได้ ≥ 200 msg/s และ alert เมื่อ consumer lag เข้าใกล้ 60 s
  - *poison message:* ส่งเข้า dead-letter queue และ alert
  - *trace ขาดช่วงข้าม broker:* `checkoutservice` ใช้ OpenTelemetry อยู่แล้ว (`initTracing` บรรทัด 154–178, propagator บรรทัด 135–137) ให้ใส่ trace context ไว้ใน message header [Inferred]

## Validation

- **Load test** ที่ 200 orders/s (เช่นใช้ Locust `loadgenerator` แบบเดียวกับ ADR-0001) โดยใช้ MailFast stub ที่จำลอง p95 1.5 s, p99 8 s, ล้ม 0.5 % และจำกัด 50 req/s แล้วตรวจว่า:
  - p95 ของ `PlaceOrder` ≤ 500 ms
  - เวลาจาก order ถึงได้ loyalty points ≤ 60 s สำหรับ**ทุก** order และจำนวน record แต้มเท่ากับจำนวน `orderId` ที่ไม่ซ้ำ (ไม่มีแต้มซ้ำ)
  - อัตราเรียก MailFast ไม่เกิน 50/s และทุกอีเมลถูกส่งสำเร็จหรืออยู่ใน dead-letter queue (ไม่มีหายเงียบ)
- **Fault injection:** ปิด `emailservice`, `loyaltyservice`, consumer ของ analytics และ MailFast stub ทีละตัว order ต้องสำเร็จ 100 % และเมื่อเปิดกลับมา backlog ต้องถูกเคลียร์จนหมด
- **Broker failure test:** kill RabbitMQ node หนึ่งตัวระหว่าง load test แล้ววัดว่ามี event หายกี่รายการ
- **Review:** ทบทวน ADR นี้หลังเทศกาล 11.11 ครั้งแรก และก่อนเพิ่ม action ใหม่ของปีหน้า (โดยเฉพาะ fraud scoring)

## Open questions

1. ช่วง peak 11.11 ยาวนานเท่าไร และอีเมลยืนยันช้าได้มากที่สุดเท่าไร? (backlog เพิ่ม 9,000 ฉบับต่อนาที ถ้ารับไม่ได้ต้องเจรจาเพิ่ม rate limit กับ MailFast ซึ่งกระทบ cost)
2. MailFast: เมื่อเกิน 50 emails/s จะปฏิเสธหรือ queue ไว้เอง? รองรับ idempotency key หรือไม่? ใช้ protocol อะไร (HTTP API หรือ SMTP)? ค่าใช้จ่ายเท่าไร?
3. ถ้า RabbitMQ ล่มตอน publish ยอมให้ event (และ loyalty points) หายได้หรือไม่ หรือต้องทำ transactional outbox (ต้องเพิ่ม datastore ให้ `checkoutservice`)?
4. Analytics: ปลายทางคืออะไร (เครื่องมือหรือคลังข้อมูลของ marketing)? event ต้องมี field อะไร มี PII ได้ไหม? ยอมให้หายหรือซ้ำได้ไหม? ต้อง replay ย้อนหลังไหม (ถ้าต้อง ควรพิจารณา Option 4 ใหม่)?
5. `loyaltyservice`: ใช้ datastore อะไร กติกาการคิดแต้มเป็นอย่างไร และรับได้กี่ msg/s ต่อ replica?
6. จะแก้ `emailservice` เดิมให้เป็น consumer ที่เรียก MailFast หรือสร้าง service ใหม่? (ภาษาและโครงสร้างของ `emailservice`: Not evidenced in the provided files)
7. Fraud scoring ของปีหน้าต้องบล็อก order ก่อนตัดเงินหรือไม่? ถ้าใช่ ต้องเป็น synchronous และอยู่นอกขอบเขตของ ADR นี้
8. latency ของการ publish ไปยัง RabbitMQ ใน cluster และงบประมาณทรัพยากร/ค่าใช้จ่ายของ RabbitMQ cluster: ยังไม่มีข้อมูล ต้องวัดและประเมิน
9. ใครเป็นผู้อนุมัติ (Deciders) และใครเป็น on-call ของ RabbitMQ?
10. `frontend` ตั้ง deadline ให้กับการเรียก `PlaceOrder` หรือไม่: Not evidenced in the provided files
