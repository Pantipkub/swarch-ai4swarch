_Agent: Claude Code · Model: Claude Opus 5.5 (`claude-opus-5-5`)_

# ADR-0002: ใช้ event `OrderPlaced` แบบ publish/subscribe ผ่าน message broker สำหรับ post-order actions

- **Status:** Proposed
- **Date:** 2026-10-27
- **Deciders:** ยังไม่ระบุ (ดู Open questions)

> หมายเหตุ: ADR นี้เขียนแบบ zero-shot (`run: a`, `context: none`) จากข้อความ requirement เท่านั้น ไม่ได้อ่านโค้ดหรือเอกสารใดของ Online Boutique ดังนั้นข้อเท็จจริงเกี่ยวกับระบบปัจจุบันถือเป็น "Not evidenced in the provided files" และข้อความที่เป็นการอนุมานติดป้าย **[Inferred]**

## Context and problem

ตาม requirement หลังจาก order ถูกสร้าง (order placed) Online Boutique ต้องทำ 3 อย่าง:

| # | Post-order action | สิ่งที่เปลี่ยน | แหล่งที่มา |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อผ่าน **MailFast** (external email provider) | แทนที่ mock logic ใน `emailservice` | requirement |
| 2 | ให้ **loyalty points** | สร้าง `loyaltyservice` ใหม่ | requirement |
| 3 | บันทึก analytics event "order placed" ให้ทีม marketing | ปลายทางยังไม่ระบุ | requirement |

- ทีม marketing คาดว่าปีหน้าจะมี post-order actions เพิ่ม (SMS, fraud scoring, การอัปเดต recommendations) → จำนวนผู้รับเหตุการณ์ "order placed" จะเพิ่มขึ้นเรื่อย ๆ
- ปัญหาที่ต้องตัดสินใจ: **บริการที่ place order จะส่งต่องานหลัง order ไปยังผู้รับหลายรายอย่างไร** โดยไม่ต้องแก้บริการนั้นทุกครั้งที่มี action ใหม่ และไม่ให้ความล้มเหลวของ external provider (MailFast) หรือบริการใหม่กระทบการสั่งซื้อ
- บริการที่ place order น่าจะเป็น `checkoutservice` **[Inferred]**; รูปแบบการเรียก `emailservice` ในปัจจุบัน (sync/async), protocol และการมี order database: Not evidenced in the provided files
- ตัวเลข (ปริมาณ order, latency target, SLA ของ MailFast, งบประมาณ): ไม่ได้ให้มา → ดู Open questions

## Decision drivers

1. **Modifiability** — เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยไม่ต้องแก้และ redeploy บริการที่ place order
2. **Availability** — ความล้มเหลวหรือความช้าของ MailFast, `loyaltyservice` หรือ analytics ต้องไม่ทำให้การสั่งซื้อล้มเหลว
3. **Performance** — latency ของ checkout ไม่ควรเพิ่มตามจำนวน post-order actions (ยังไม่มีค่า target)
4. **Reliability** — ทุก action ต้องได้รับเหตุการณ์ ไม่หาย และ loyalty points ต้องไม่ถูกให้ซ้ำ
5. **Integrability / security** — แยก dependency กับ MailFast (API contract, credentials) ไว้ที่เดียว
6. **Cost / operability** — ต้นทุนและภาระการดูแล infrastructure ใหม่ (ยังไม่ทราบงบและทักษะทีม)

## Considered options

1. **A — Synchronous direct calls:** บริการที่ place order เรียก `emailservice` (MailFast), `loyaltyservice` และ analytics โดยตรงทีละตัวก่อนตอบผู้ใช้
2. **B — Event-driven publish/subscribe (choreography):** บริการที่ place order publish event `OrderPlaced` หนึ่งครั้งไปยัง message broker; `emailservice`, `loyaltyservice` และ analytics consumer subscribe แยกกันและประมวลผลแบบ asynchronous
3. **C — Post-order orchestrator service:** สร้างบริการใหม่ (เช่น `postorderservice`) ที่รับคำขอหนึ่งครั้งจากบริการที่ place order แล้วเรียกแต่ละ action ตามลำดับ พร้อม retry ที่ศูนย์กลาง

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A — Synchronous direct calls | เรียบง่าย ไม่ต้องมี infrastructure ใหม่; รู้ผลทันทีว่าแต่ละ action สำเร็จหรือไม่; debug ง่าย | ทุก action ใหม่ต้องแก้บริการที่ place order (modifiability ต่ำ); latency ของ checkout เท่ากับผลรวมของทุก call; MailFast ล่มหรือช้าทำให้ checkout ล้มหรือช้าตาม (availability ต่ำ); retry/timeout ทั้งหมดกองอยู่ในบริการเดียว |
| B — Event-driven pub/sub | เพิ่ม action ใหม่ = เพิ่ม subscriber ไม่ต้องแก้ publisher (modifiability สูง); checkout ทำงานต่อได้แม้ consumer ล่ม (availability); checkout เพิ่ม latency แค่การ publish หนึ่งครั้ง; consumer แต่ละตัว scale และ retry ได้อิสระ; broker เก็บ event ไว้ replay ได้ | ต้องมี message broker ใหม่ให้ deploy และดูแล (cost, operability); eventual consistency — อีเมลและ points มาช้ากว่า order; delivery แบบ at-least-once บังคับให้ consumer ต้อง idempotent; trace การไหลข้ามบริการยากขึ้น; ต้องจัดการ schema evolution ของ event |
| C — Orchestrator service | ลำดับและสถานะของ post-order flow อยู่ที่เดียว มองเห็นง่าย; retry/compensation รวมศูนย์; บริการที่ place order ไม่ต้องรู้จักแต่ละ action | ทุก action ใหม่ยังต้องแก้ orchestrator (modifiability ปานกลาง); ถ้าเรียกแบบ sync orchestrator กลายเป็นจุดล้มเดี่ยวใหม่ของ checkout; ถ้าต้องการ async ก็ยังต้องมี queue อยู่ดี; มีบริการเพิ่มอีกหนึ่งตัวที่ต้องดูแล |

## Decision

เลือก **Option B — Event-driven publish/subscribe ด้วย event `OrderPlaced` ผ่าน message broker**

- ตอบ driver อันดับ 1 (modifiability) ได้ดีที่สุด: action ของปีหน้า (SMS, recommendations) เพิ่มเป็น subscriber ใหม่ได้โดยไม่แตะบริการที่ place order
- ตอบ driver อันดับ 2–3 (availability, performance): checkout ไม่ต้องรอ MailFast, `loyaltyservice` หรือ analytics
- โครงสร้างที่ตัดสินใจ:
  - บริการที่ place order (`checkoutservice` **[Inferred]**) publish `OrderPlaced` หนึ่งครั้งหลัง order สำเร็จ โดยมี order ID เป็น idempotency key
  - `emailservice` เป็น consumer และเป็น adapter (anti-corruption layer) ไปยัง MailFast แทน mock logic เดิม; credentials ของ MailFast อยู่ที่ `emailservice` เท่านั้น
  - `loyaltyservice` (ใหม่) เป็น consumer ที่ให้ points แบบ idempotent ตาม order ID
  - analytics consumer ส่ง event ไปยังปลายทางของทีม marketing (ปลายทางยังไม่ระบุ)
  - consumer แต่ละตัวมี subscription/queue ของตัวเอง พร้อม retry with backoff และ dead-letter queue
- ยังไม่เลือกเทคโนโลยี broker (เช่น RabbitMQ, Kafka หรือ managed cloud service) เพราะไม่มีข้อมูลปริมาณ งบ และทักษะทีม → Open questions
- ข้อยกเว้นที่ต้องทบทวน: ถ้า fraud scoring ของปีหน้าต้อง **บล็อก** order ก่อนยืนยัน จะไม่ใช่ post-order action แบบ async และต้องทำ ADR แยก

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่โดยเพิ่ม subscriber เท่านั้น ไม่ต้องแก้หรือ redeploy publisher (quality attribute: modifiability)
- **Positive:** MailFast หรือ `loyaltyservice` ล่มไม่ทำให้การสั่งซื้อล้มเหลว event รออยู่ใน broker และถูกประมวลผลเมื่อบริการกลับมา (quality attribute: availability)
- **Positive:** latency ของ checkout ไม่เพิ่มตามจำนวน actions (quality attribute: performance)
- **Positive:** dependency กับ MailFast อยู่ใน `emailservice` ที่เดียว เปลี่ยน provider อีกครั้งได้โดยไม่กระทบบริการอื่น (quality attribute: modifiability, security)
- **Negative:** ต้องมี message broker ใหม่ให้ deploy, monitor และ upgrade (quality attribute: cost, operability)
- **Negative:** อีเมลยืนยันและ points มาถึงแบบ eventual ผู้ใช้อาจไม่เห็นทันทีหลัง checkout (quality attribute: consistency, usability)
- **Negative:** debug การไหลข้ามหลายบริการยากขึ้น ต้องมี distributed tracing และ correlation ID (quality attribute: observability, testability)
- **Risks and mitigations:**

| Risk | Mitigation |
|---|---|
| Event ซ้ำ (at-least-once) → ให้ points หรือส่งอีเมลซ้ำ | consumer idempotent โดยใช้ order ID เป็น key |
| Order สำเร็จแต่ publish event ไม่สำเร็จ → อีเมล/points หาย | transactional outbox ถ้าบริการที่ place order มี database (Not evidenced in the provided files) หรือ retry การ publish พร้อม alert |
| MailFast ล่ม ช้า หรือ rate-limit | retry with backoff + dead-letter queue + alert ที่ `emailservice`; ค่า rate limit ยังไม่ทราบ |
| Broker ล่ม → publish ไม่ได้ | ใช้ broker แบบ replicated หรือ managed (เลือกตามงบ); ต้องตัดสินว่า checkout จะ fail หรือ buffer (Open question) |
| Schema ของ `OrderPlaced` เปลี่ยนแล้ว consumer พัง | versioned schema, เพิ่ม field แบบ backward-compatible, contract test ระหว่าง publisher กับ consumer |
| ข้อมูลส่วนบุคคล (อีเมล, ที่อยู่) อยู่ใน event ที่ทุก consumer อ่านได้ | ใส่เฉพาะ field ที่จำเป็น; ส่งให้ analytics แบบไม่มี PII; จำกัดสิทธิ์การ subscribe |

## Validation

- **Load test:** วัด latency ของ checkout ก่อนและหลังเปลี่ยน; เกณฑ์ผ่านยังไม่มีตัวเลข (Open question)
- **Fault injection:** ปิด MailFast (ใช้ stub), `loyaltyservice` และ analytics consumer ระหว่าง checkout → order ต้องสำเร็จ และเมื่อเปิดกลับมา event ต้องถูกประมวลผลครบ
- **Duplicate test:** ส่ง `OrderPlaced` เดิมซ้ำ → points ถูกให้ครั้งเดียว อีเมลถูกส่งครั้งเดียว
- **Modifiability check:** เพิ่ม subscriber ตัวอย่าง (เช่น SMS stub) แล้วยืนยันว่าไม่ต้องแก้ publisher หรือ consumer เดิม
- **Metrics:** consumer lag, จำนวน message ใน dead-letter queue, อัตราการส่งสำเร็จผ่าน MailFast, เวลาจาก order ถึงอีเมลถึงผู้ใช้
- **Review date:** ยังไม่ระบุ — ควรทบทวนก่อนเพิ่ม action ของปีหน้า โดยเฉพาะ fraud scoring

## Open questions

1. ใครเป็น Deciders ของ ADR นี้?
2. ปริมาณ order (เฉลี่ยและ peak) และ latency target ของ checkout คือเท่าไร?
3. ยอมรับได้ไหมที่อีเมลยืนยันและ points มาถึงแบบ eventual และต้องถึงภายในเวลาเท่าไร?
4. ถ้าส่งอีเมลยืนยันไม่สำเร็จ order ยังถือว่าสำเร็จหรือไม่? ถ้า publish event ไม่ได้ checkout ควร fail หรือ buffer?
5. MailFast: รูปแบบ API (REST, SMTP, …), rate limit, SLA, ราคา, วิธี authentication, มี sandbox สำหรับทดสอบหรือไม่, ข้อกำหนด data residency/PDPA?
6. กฎการคำนวณ loyalty points คืออะไร ต้องแสดงให้ลูกค้าเห็นทันทีหลัง checkout หรือไม่ และทีม/ภาษาใดจะพัฒนา `loyaltyservice`?
7. Analytics: ปลายทางคืออะไร (เครื่องมือหรือ data warehouse ใด), ต้องการ field อะไร, รับ PII ได้หรือไม่, ยอมรับ delay ได้เท่าไร?
8. Message broker: มี broker หรือ infrastructure เดิมอยู่แล้วหรือไม่, ทีมมีทักษะกับเทคโนโลยีใด, ใช้ managed cloud service ได้หรือไม่, งบประมาณเท่าไร?
9. บริการที่ place order คือ `checkoutservice` จริงหรือไม่ และมี database สำหรับ transactional outbox หรือไม่? (Not evidenced in the provided files)
10. Fraud scoring ของปีหน้าต้องบล็อก order ก่อนยืนยันหรือไม่? ถ้าใช่ ต้องอยู่ใน synchronous path และต้องทำ ADR แยก
11. ต้องเก็บ event ไว้ replay นานเท่าไร (retention) เพื่อให้ subscriber ใหม่ในปีหน้า backfill ข้อมูลย้อนหลังได้หรือไม่?
12. ระบบมี distributed tracing และ monitoring อยู่แล้วหรือไม่? (Not evidenced in the provided files)
