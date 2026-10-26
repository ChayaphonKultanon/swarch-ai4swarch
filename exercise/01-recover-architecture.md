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
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR=paymentservice:50051`) | ✅ | env var อยู่ที่ `checkoutservice.yaml` L61–62 ส่วน `src/checkoutservice/main.go` L116 อ่านค่าด้วย `mustMapEnv` และ L370 เรียก `PaymentService.Charge` จริง แต่ `src/paymentservice/charge.js` L56 ระบุว่า "(pretend) charges the card" และ L85 คืน `uuidv4()` แบบสุ่ม จึงเป็นการตัดเงิน**จำลอง** |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR=emailservice:5000`) | ✅ | env var อยู่ที่ `checkoutservice.yaml` L63–64 และ `main.go` L380 เรียก `SendOrderConfirmation` หลังจาก ship order แล้ว พอร์ตตรงกับ `emailservice.yaml` L82–83 (5000 → 8080) แต่ `src/emailservice/email_server.py` L124 จะ raise `'non-dummy mode not implemented yet'` และ L200 รันด้วย `dummy_mode = True` เสมอ จึงไม่มีการส่งอีเมลจริง |
| `frontend` → `shippingservice` | `frontend.yaml` (`SHIPPING_SERVICE_ADDR=shippingservice:50051`) | ✅ | env var อยู่ที่ `frontend.yaml` L77–78 และ `src/frontend/main.go` L137 ส่วน `src/frontend/rpc.go` L88 เรียก**เฉพาะ** `GetQuote` โดย `ShipOrder` ถูกเรียกจาก checkout เท่านั้น (`checkoutservice/main.go` L387) จึงตรงกับ label "Gets shipping quote" |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR=productcatalogservice:3550`) | ✅ | env var อยู่ที่ `recommendationservice.yaml` L60–61 ส่วน `src/recommendationservice/recommendation_server.py` L131 อ่าน env และ L73 เรียก `ListProducts` |
| `cartservice` → `redis-cart` | `cartservice.yaml` (`REDIS_ADDR=redis-cart:6379`) | ✅ | env var อยู่ที่ `cartservice.yaml` L50–51 และ `src/cartservice/src/Startup.cs` L29 อ่าน `REDIS_ADDR` ส่วน L34–40 จะใช้ `RedisCartStore` เมื่อมีค่านี้ ถ้าไม่มีจะใช้ Spanner, AlloyDB หรือ in-memory แทน (L42–56) |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
  - **Dependency ที่ไม่มีจริง:** ไม่พบ ทั้ง 15 edge ระหว่าง container (ไม่นับ Customer และ `loadgenerator`) มีโค้ดฝั่งผู้เรียกที่อ่าน env var และสร้าง client ไปยังผู้ถูกเรียกจริง (`frontend/main.go` L132–138, `checkoutservice/main.go` L111–116, `recommendation_server.py` L131, `cartservice/src/Startup.cs` L29)
  - **Dependency ที่ขาดไป:** `frontend` → `shoppingassistantservice` (HTTP) มีอยู่ในโค้ดจริง (`frontend/main.go` L139 ใช้ `mustMapEnv` ส่วน `handlers.go` L464 ส่ง `POST http://…`) และมีซอร์สใน `src/shoppingassistantservice/` แต่ไม่มี manifest การที่ AI ไม่วาดลงแผนภาพจึงถูกต้องสำหรับ deployment นี้ และ AI ก็บันทึกไว้ใน Gaps แล้ว
  - **Container ที่น่าสงสัย:** `loadgenerator` ถูกวางไว้ใน `System_Boundary` เหมือน container ใน production เส้นเชื่อมไป `frontend` มีอยู่จริง (`loadgenerator.yaml` L69–70) แต่เป็นเครื่องมือทดสอบที่ไม่ถูก deploy ตามค่าเริ่มต้น เพราะ `kustomization.yaml` L24 comment บรรทัด `# - loadgenerator.yaml` ไว้ ควรย้ายออกนอก boundary หรือตัดออก
  - **ข้อมูลที่ทำให้เข้าใจผิด:** `paymentservice` ("Charges credit cards") และ `emailservice` ("Order confirmation emails") ดูเหมือนเชื่อมต่อระบบภายนอกจริง แต่ทั้งคู่เป็น **mock**
  - **ข้อมูลที่ขาด:** AI ระบุว่าภาษา "not evidenced" เพราะได้อ่านเฉพาะ manifest แต่จริงๆ แล้ว `src/` แสดงว่าเป็นระบบหลายภาษา ได้แก่ Go (`frontend`, `checkoutservice`, `productcatalogservice`, `shippingservice`), C# (`cartservice`, `cartservice.csproj`), Node.js (`currencyservice`, `paymentservice`), Python (`emailservice`, `recommendationservice`, `loadgenerator`) และ Java (`adservice`, `build.gradle`)

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: จากไฟล์ `.proto` เพียงไฟล์เดียว agent ไม่สามารถรู้ได้ว่า service ใดเรียก service ใด ไม่รู้ว่ามี `frontend` เป็นทางเข้าของผู้ใช้ และไม่รู้ว่ามี data store `redis-cart` เพราะ proto ประกาศเพียงว่าแต่ละ service *ให้บริการ* อะไร ไม่ได้บอกว่าใครเป็นผู้ใช้บริการ การเชื่อมต่อจริงอยู่ใน env var `*_SERVICE_ADDR` ของ manifest ซึ่งมีให้เฉพาะใน Run B ผลคือ Run A ได้เพียง 9 container ที่ไม่มีเส้นเชื่อมเลย ส่วน Run B ได้ 15 edge ระหว่าง container ซึ่งตรวจกับโค้ดแล้วถูกต้องทุกเส้น
2. ข้อผิดพลาดที่จะทำให้สมาชิกใหม่เข้าใจผิดมากที่สุดคือการวาง `loadgenerator` ไว้ในขอบเขตระบบ เพราะแผนภาพดูเหมือนบอกว่ามี component ที่ส่ง traffic ปลอมเข้า `frontend` อยู่ใน production ตลอดเวลา ทั้งที่ `kustomization.yaml` L24 ไม่ได้ deploy ตัวนี้ คนใหม่จึงอาจตีความ traffic, metric หรือ log ผิด หรือเข้าใจผิดว่าเป็นส่วนหนึ่งของระบบที่ต้องดูแล
