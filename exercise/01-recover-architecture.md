**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `frontend` → `cartservice` | `frontend.yaml` (`CART_SERVICE_ADDR` L73-74) | ✅ | `src/frontend/main.go:134,143` อ่าน env แล้ว `mustConnGRPC`; `src/frontend/rpc.go:58,63,68` เรียก `GetCart`, `EmptyCart`, `AddItem` |
| `frontend` → `shippingservice` | `frontend.yaml` (`SHIPPING_SERVICE_ADDR` L77-78) | ✅ | `src/frontend/rpc.go:88` เรียกแค่ `GetQuote` (ไม่เรียก `ShipOrder`) ตรงกับที่ AI ใส่ไว้ใน Gaps ว่ายังไม่รู้ว่าเรียก RPC ไหน |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR` L63-64) | ✅ | `src/checkoutservice/main.go:115,122,380` เรียก `SendOrderConfirmation`; ถ้าล้มแค่ log warning (L273-277) แต่ `emailservice` เองรันใน **dummy mode** ไม่ได้ส่งอีเมลจริง (`src/emailservice/email_server.py:108-110,167`) |
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR` L61-62) | ✅ | `src/checkoutservice/main.go:116,123,370` เรียก `Charge`; `paymentservice` เป็น mock ตรวจเลขบัตรแล้วคืน uuid สุ่ม (`src/paymentservice/charge.js:59-80`) ไม่มี payment provider ภายนอก |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR` L60-61) | ✅ | `src/recommendationservice/recommendation_server.py:131-136` สร้าง `grpc.insecure_channel` + `ProductCatalogServiceStub` |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR` L50-51) | ✅ (มีเงื่อนไข) | `src/cartservice/src/Startup.cs:29` อ่าน `REDIS_ADDR` แต่ store ถูกเลือกตาม config: มี `SpannerCartStore` และ `AlloyDBCartStore` ด้วย (`Startup.cs:30-32,48-49`) |
| `loadgenerator` → `frontend` | `loadgenerator.yaml` (`FRONTEND_ADDR` L82-83) | ✅ | `src/loadgenerator/locustfile.py:35-75` ยิง HTTP GET/POST ไปที่หน้าเว็บ (`/`, `/cart`, `/cart/checkout` ฯลฯ) |

ทั้ง 17 dependency ในตารางของ Run B มีอยู่จริงในโค้ด ไม่พบ dependency ที่ไม่มีจริง

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B:
  - **ขาด `frontend` → `shoppingassistantservice` (HTTP)**: `src/frontend/handlers.go:465` ส่ง HTTP POST ไปที่ `SHOPPING_ASSISTANT_SERVICE_ADDR` (route `/bot`, `src/frontend/main.go:163`) AI เห็น env นี้แต่ใส่ไว้ใน Gaps เพราะไม่มี manifest ซึ่งระมัดระวังถูกต้อง แต่ทำให้แผนภาพขาด service จริงไปหนึ่งตัว
  - **ขาด external systems**: `shoppingassistantservice` เรียก Google Gemini (`ChatGoogleGenerativeAI`) และ AlloyDB vector store (`src/shoppingassistantservice/shoppingassistantservice.py:22-54`) แต่แผนภาพไม่มี `System_Ext` เลย
  - **ขาด dependency แบบ optional**: OpenTelemetry collector เมื่อ `ENABLE_TRACING=1` (`src/frontend/main.go:113,178`, `src/checkoutservice/main.go:90,164`); `productcatalogservice` อ่าน AlloyDB ได้ถ้าตั้ง `ALLOYDB_CLUSTER_NAME` ไม่งั้นอ่าน `products.json` (`src/productcatalogservice/catalog_loader.go:37-47`); packaging service ผ่าน HTTP (`src/frontend/packaging_info.go:54`)
  - **container ที่ทำให้เข้าใจผิด**: แผนภาพบอกว่าภาษา "not evidenced" ทุกตัว ซึ่งถูกตามหลักฐาน แต่ Dockerfile แสดงว่าระบบเป็น polyglot: Go (`frontend`, `checkoutservice`, `productcatalogservice`, `shippingservice`), C# (`cartservice`), Node.js (`currencyservice`, `paymentservice`), Python (`emailservice`, `recommendationservice`, `loadgenerator`), Java (`adservice`) และ `emailservice`/`paymentservice` เป็น mock ไม่ใช่ระบบจริง

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. **เปรียบเทียบ A กับ B:** จาก `.proto` ไฟล์เดียว agent ไม่รู้เลยว่า**ใครเรียกใคร** (Run A มี 0 dependency) ไม่รู้ว่ามี `frontend` เป็นทางเข้า และไม่รู้ว่ามี `redis-cart` เพราะ `.proto` บอกแค่ว่าแต่ละ service *ให้บริการ* อะไร ไม่ได้บอกว่า *ใช้* อะไรหรือ deploy อย่างไร พอได้ manifest ใน Run B ตัวแปร `*_ADDR` ทำให้เห็น wiring ทั้งหมดและ dependency ทุกเส้นถูกต้อง
2. **ข้อผิดพลาดที่อันตรายที่สุด:** แผนภาพ Run B บอกว่า**ไม่มี external system** ทั้งที่ `frontend` ส่งข้อความแชตของผู้ใช้ไปยัง `shoppingassistantservice` ซึ่งส่งต่อไปยัง Google Gemini นอกระบบ สมาชิกใหม่ที่ใช้แผนภาพนี้ทำ security/privacy review จะไม่รู้ว่ามีข้อมูลผู้ใช้ออกนอกระบบ
