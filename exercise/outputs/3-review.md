ผู้รีวิว: Claude Code (agent) · model Claude Opus 5.5 (`claude-opus-5-5`)

# Review: ADR-0002 (RabbitMQ + transactional outbox สำหรับงานหลังสั่งซื้อ)

หลักฐานที่ใช้มีแค่ `docs/adr/0002-use-rabbitmq-for-post-order-actions.md` (อ้างเป็น "ADR Lxx") และ `docs/scenario.md`

## 1. Quality-attribute scenarios

| # | Source | Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|---|
| QAS-1 Performance | ลูกค้าช่วง 11.11 | `PlaceOrder` 200 req/s | peak, คิวอีเมลค้างหลายหมื่นฉบับ และ relay กำลังระบาย outbox | เขียน `orders` + `outbox` ใน transaction เดียวแล้วตอบทันที ไม่รอ MailFast หรือ RabbitMQ | p95 ของ `PlaceOrder` ≤ 500 ms (ส่วนที่เพิ่มต้อง ≤ ~200 ms, ADR L26) |
| QAS-2 Reliability | RabbitMQ ทั้ง cluster | ล่ม 2 นาที (ผู้รีวิวสมมติตัวเลขเอง) | 200 orders/s ทำให้ outbox สะสม ~24,000 row | order ยังสำเร็จ event รอใน outbox แล้วส่งเมื่อ broker กลับมา | order สำเร็จ 100 %, แต้มหาย 0 และซ้ำ 0; order หลัง broker กลับมาได้แต้มใน ≤ 60 s; order ในช่วงที่ล่มต้องระบุว่าเป็นข้อยกเว้นหรือไม่ (ADR ยังไม่ระบุ) |
| QAS-3 Throughput ของอีเมล | MailFast | 200 orders/s นาน 10 นาที ขณะที่ MailFast รับได้ 50/s, ล้ม 0.5 % และ p99 8 s | peak | `order.email` เก็บ backlog ไว้ และ consumer ส่ง ≤ 50/s พร้อม retry และ DLQ | อีเมลหาย 0 (ส่งสำเร็จหรืออยู่ใน DLQ), อัตราเรียก ≤ 50/s และ backlog 90,000 ฉบับเคลียร์ได้ในราว 30 นาทีหลัง peak (ADR L95) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | ข้ออ้างว่า "event จะหายก็ต่อเมื่อ `checkout-db` และ RabbitMQ ล่มพร้อมกัน" (L73) ไม่จริง เพราะยังหายได้เมื่อ (ก) pod ของ `checkoutservice` crash หลัง `chargeCard`/`shipOrder` แต่ก่อน commit (ข) fallback publish แบบ best-effort ล้มเหลว (ค) PostgreSQL แบบ primary/replica (L99) failover ไปยัง replica ที่ replicate แบบ async (ง) queue ไม่ได้ replicate (ADR ไม่ระบุ quorum/classic) | ลูกค้าถูกตัดเงินแต่ไม่ได้แต้มหรืออีเมล ขัดกับ "exactly once" ใน `docs/scenario.md` (Loyalty points) และไม่มี row ใน `orders` ให้ reconcile |
| Risk | relay เป็น goroutine ใน pod เดียวกับ `PlaceOrder` และใช้ `checkout-db` ตัวเดียวกัน (L76) | หลัง broker กลับมา relay ต้องระบาย ~24,000 row (QAS-2) ไปพร้อมกับ order ใหม่ 200/s จึงแย่ง CPU, connection และ IO กับ checkout และเสี่ยงทำให้ p95 เกิน 500 ms |
| Sensitivity point | concurrency/prefetch ของ consumer อีเมล และกลไกของ "rate limiter ตัวเดียว" ข้าม replica (L79) | ถ้าจะได้ 50/s ที่ latency ราว 1.5 s ต้องมี call ค้างพร้อมกันราว 75 call (Little's law, ใช้ p95 แทนค่าเฉลี่ยแบบหยาบ **[Inferred]**) และ p99 8 s ทำให้ต้องมีมากกว่านั้น ถ้าตั้งไว้ 10 จะได้แค่ ~7/s และ backlog ใช้เวลาหลายชั่วโมงแทน 30 นาที |
| Sensitivity point | latency ของ commit ใน `checkout-db` และโหมด replication (sync หรือ async) | เป็นต้นทุนใหม่เพียงตัวเดียวบน path ของ `PlaceOrder` และยังไม่มีตัวเลข (Open question 3) แบบ sync ปลอดภัยกว่า (แก้ข้อ ค) แต่กินงบ 200 ms มากกว่า |
| Sensitivity point | เวลา outage/failover ของ RabbitMQ และ polling interval ของ relay | outbox ทำให้ event "ไม่หาย" แต่ไม่ได้ทำให้ "ทันเวลา" ถ้า broker ล่มนานกว่าราว 1 นาที order ในช่วงนั้นจะได้แต้มเกิน 1 นาที ขณะที่ L62 เขียนไว้แค่ว่า "แค่ล่าช้า" |
| Trade-off point | เพิ่ม `checkout-db` เพื่อทำ outbox | reliability ดีขึ้น แต่ performance (DB write ทุก order) และ operability/cost แย่ลง เพราะทีม 4 คนต้องดูแล stateful component ใหม่ 2 ตัว (L94) ก่อน 11.11 ประสบการณ์ PostgreSQL ของทีม: Not evidenced in the provided files และยังไม่มีเจ้าของ on-call (Open question 8) |
| Trade-off point | fallback ไป publish ตรงเมื่อเขียน DB ไม่ได้ (L73) | availability ของ checkout ดีขึ้น แต่ดึง RabbitMQ กลับเข้า critical path (ขัดกับ L62 และ L88) และ order กลุ่มนี้ไม่มี row ใน `orders` ทำให้ reconcile (L90, L113) มองไม่เห็น ทั้งที่ ADR ชี้เองว่าโค้ดเดิมไม่มี timeout (L17) แต่ไม่กำหนด timeout ให้ DB write และ fallback |
| Non-risk | แยก queue ต่อ consumer (L78–81) | backlog อีเมล 90,000 ฉบับไม่ถ่วงแต้ม ส่วน `loyaltyservice` รับแค่ 200 msg/s |
| Non-risk | outbox + publisher confirms (at-least-once) และ `orderId` เป็น unique key โดย ack หลัง commit (L74, L80) | เมื่อ event ถึง broker แล้ว จะได้ผลแบบ exactly once ที่ consumer |
| Non-risk | ตัด Option 1 ด้วยตัวเลข (L59) และไม่มี `credit_card` ใน payload (L77) | ตรวจสอบได้กับ `docs/scenario.md` (p95 1.5 s > 500 ms และ 200 > 50/s) และจำกัด PII ที่ส่งไปยัง consumer |

## 3. Missing or weak

- **Options not considered:** (1) ใช้ `checkout-db` เป็น queue ให้ consumer อ่าน outbox เองด้วย `SKIP LOCKED` โดยไม่มี RabbitMQ ซึ่งลด stateful component เหลือตัวเดียว แต่แลกกับการที่ consumer ผูกกับ schema ของ checkout และต้องทำ retry/DLQ เอง ADR ควรอธิบายว่าทำไมยังคุ้มที่จะมี RabbitMQ เมื่อมี DB แล้ว (2) เขียน order เป็น `pending` ก่อน `chargeCard` เพื่อปิดช่อง crash (ก) โดยแลกกับ DB write 2 ครั้ง
- **Consequences not stated:** `checkout-db` กลายเป็นที่เก็บ PII แห่งใหม่ (`email` และที่อยู่ใน `orders`/`outbox`, L77) แต่ไม่ระบุ retention และการเข้ารหัส backup (driver 6); ภาระ DB ราว 4 writes ต่อ order (insert `orders`, insert/update/delete `outbox`) หรือประมาณ 800 writes/s ที่ peak **[Inferred]**; ไม่ระบุชนิดของ queue และจำนวน node ทั้งที่ต้องเก็บ ≥ 90,000 ข้อความ
- **Unsupported or likely wrong:** L73 (หายเฉพาะเมื่อล่มพร้อมกัน), L62/L88 ("ไม่ต้องรอแม้แต่ RabbitMQ") ไม่จริงเมื่อเข้า fallback path; L63 บอกว่า "ทีมไม่มีประสบการณ์ Kafka ตาม scenario" แต่ `docs/scenario.md` บอกเพียงว่า 2 คนเคยใช้ RabbitMQ และไม่ได้พูดถึง Kafka; Validation (L113) ไม่ทดสอบกรณี `checkout-db` ล่ม, pod crash ระหว่าง charge กับ commit และ DB กับ broker ล่มพร้อมกัน; เลขบรรทัดใน `system/online-boutique/src/checkoutservice/main.go` ไม่ได้ตรวจในรีวิวนี้ เพราะ task อนุญาตให้อ่านเฉพาะ ADR และ scenario

## 4. Verdict: **Accept with changes**

1. **แก้ข้ออ้างเรื่อง event ไม่หาย:** ระบุ loss window ทุกกรณี (ก–ง), ตัดสินใจว่าจะเก็บ fallback ไว้หรือไม่ (ถ้าเก็บ ต้อง backfill `orders` ภายหลัง) และกำหนด timeout ของ DB write กับ fallback ให้อยู่ในงบ ~200 ms
2. **ระบุ config ที่การรับประกันนี้ต้องพึ่ง:** quorum queues บน RabbitMQ ≥ 3 node, โหมด replication ของ `checkout-db` และผูกเวลา failover ของทั้งสองตัวเข้ากับงบ loyalty 60 s (alert 30 s, L103) รวมถึงระบุว่า order ในช่วง outage เป็นข้อยกเว้นหรือไม่
3. **กำหนดขนาดและทดสอบให้ครบ:** ระบุ concurrency ของ consumer อีเมลและกลไก global limiter 50/s และเพิ่ม fault injection สำหรับ `checkout-db` ล่ม, kill pod ระหว่าง `PlaceOrder` และการที่ relay ระบาย backlog ขณะมี order 200/s โดยวัด p95 ไปด้วย
