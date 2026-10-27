**ภาษา:** ภาษาไทย | [English](00-prompt-log.en.md)

# บันทึกการใช้ prompt

**ชื่อ / รหัสนิสิต:** Piyapanchanok Meeboonsalang / 6630196021
**agent ที่ใช้:** Claude Code (Claude desktop app, Code tab) แต่ละ run รันเป็น subagent ที่เริ่ม context ใหม่ (เทียบเท่า session ใหม่) และถูกสั่งให้อ่านเฉพาะไฟล์ที่อนุญาต
**โมเดล:** Claude Opus 5.5 (`claude-opus-5-5`)
**แพ็กเกจที่ใช้:** บัญชี Claude ส่วนตัว

บันทึกทุกคำขอที่ส่งให้ agent ช่อง "ไฟล์ที่ agent อ่าน" คือไฟล์ที่ agent รายงาน หรือที่นิสิตเห็นว่า agent เปิด

| # | งาน / run | agent + โมเดล | ข้อความที่ส่ง (ย่อ) | ไฟล์ที่ agent อ่าน | อยู่ในขอบเขตหลักฐานที่อนุญาตหรือไม่ | มีประโยชน์ (1–5) | หมายเหตุ |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Claude Code + Opus 5.5 | `prompts/1-…` run A, evidence `protos/demo.proto` | `AGENTS.md`, `prompts/1-recover-architecture.md`, `protos/demo.proto` | ✅ | 2 | ได้ 9 containers แต่ 0 dependency และติด [Inferred] ให้ gRPC อย่างซื่อตรง ใช้ประโยชน์ได้น้อยเพราะหลักฐานน้อย |
| 2 | 1-B | Claude Code + Opus 5.5 | `prompts/1-…` run B, evidence `demo.proto` + `kubernetes-manifests/` | `AGENTS.md`, `prompts/1-…`, `demo.proto`, manifest ทั้ง 12 ไฟล์ + `kubernetes-manifests/README.md` (list โฟลเดอร์ด้วย Glob) | ✅ | 4 | 12 containers, 17 dependencies ถูกทั้งหมด แต่ขาด `shoppingassistantservice` และ external systems (ดูงานที่ 1) |
| 3 | 2-a | Claude Code + Opus 5.5 | `prompts/2-…` run a, context none, R-12, Thai | `AGENTS.md`, `prompts/2-write-adr.md`, `docs/adr/template.md` | ✅ (ไม่ได้เปิด `docs/scenario.md`) | 3 | เลือก pub/sub แต่ไม่เลือกเทคโนโลยี broker ไม่มีตัวเลข; Open questions 12 ข้อ |
| 4 | 2-b | Claude Code + Opus 5.5 | `prompts/2-…` run b, context scenario + ADR-0001 + `checkoutservice/main.go`, R-12, Thai | `AGENTS.md`, `prompts/2-…`, `docs/adr/template.md`, `docs/scenario.md`, `docs/adr/0001-…`, `src/checkoutservice/main.go` | ✅ | 5 | ใช้ตัวเลข scenario ครบ เลือก RabbitMQ เลขบรรทัดที่อ้างถูกต้อง; ใช้เป็นฐานของ ADR-0002 |
| 5 | 3 | Claude Code + Opus 5.5 | `prompts/3-…`, adr `docs/adr/0002-use-rabbitmq-for-post-order-actions.md`, Thai | `AGENTS.md`, `prompts/3-review-adr.md`, `docs/adr/0002-…`, `docs/scenario.md` | ✅ (ไม่ได้แก้ ADR) | 5 | Accept with changes; จับได้ว่าข้ออ้าง "event หายเฉพาะเมื่อ DB และ broker ล่มพร้อมกัน" ผิด และประโยค "ทีมไม่มีประสบการณ์ Kafka ตาม scenario" ไม่มีใน scenario |
| 6 | ช่วยตรวจสอบ (ไม่ใช่ run) | Claude Code + Opus 5.5 (session หลัก) | ขอให้ช่วยร่างตารางตรวจสอบในงาน 1, 2 และแก้ ADR-0002 | `src/*` (เช่น `frontend/rpc.go`, `frontend/handlers.go`, `checkoutservice/main.go`, `recommendation_server.py`, `Startup.cs`, `locustfile.py`, Dockerfile ทุกตัว), `kubernetes-manifests/` ไม่ได้อ่าน `docs/review-board/` | นอกขอบเขตของ run (ตั้งใจ เป็นการตรวจสอบ) | | **ส่วนที่ใบงานให้นิสิตทำเองถูกร่างโดย AI** นิสิตต้องตรวจและเขียนใหม่ด้วยตัวเอง |
| 7 | ส่วน B | Claude Code + Opus 5.5 (session หลัก) | (1) ก่อนประชุม: ขอให้จำลองการประชุมและร่างสคริปต์ไว้ซ้อม (2) หลังประชุม: ขอให้แปลบันทึกการประชุมของกลุ่ม (ภาษาอังกฤษ) เป็นภาษาไทย แล้วกรอกใบงาน 04 | `docs/review-board/` (README + การ์ด P, S, X), `exercise/04-review-board.md`, `docs/adr/0002-…`, `exercise/outputs/3-review.md` | ❌ `AGENTS.md` ห้ามอ่าน `docs/review-board/` | | ผลการจำลองไม่ได้ใช้ในใบงานที่ส่ง ใบงาน 04 มาจากบันทึกของกลุ่ม (รีวิว ADR ของ Pitchanan) AI ทำแค่แปลและจัดรูปแบบ |
| 8 | สะท้อนคิด 05 | Claude Code + Opus 5.5 (session หลัก) | ขอให้แปล Reflect ภาษาอังกฤษ ที่ผมส่งให้เป็นภาษาไทย | `exercise/05-reflection.md` | ✅ | | AI ทำแค่แปล ร่างก่อนหน้าของ AI ถูกแทนที่ทั้งหมด |
