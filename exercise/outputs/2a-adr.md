> Agent: Claude Code (VS Code extension) · Model: Claude Opus 5.5 (`claude-opus-5-5`) · Run: a (zero-shot, `context: none`)

# ADR-0002: ใช้ event `OrderPlaced` ผ่าน message broker เพื่อทำ post-order actions

- **Status:** Proposed
- **Date:** 2026-10-26
- **Deciders:** ไม่ได้ระบุไว้ใน requirement (ดู Open questions)

## Context and problem

หลังจากลูกค้าสั่งซื้อสำเร็จ Online Boutique ต้องทำงานต่อจากนี้ (จาก requirement):

| # | Post-order action | ปลายทาง | หมายเหตุ |
|---|---|---|---|
| 1 | ส่งอีเมลยืนยันคำสั่งซื้อ | **MailFast** (external email provider รายใหม่) | แทนที่ logic จำลอง (mock) ใน `emailservice` |
| 2 | ให้ **loyalty points** | `loyaltyservice` (ยังไม่มี ต้องสร้างใหม่) | |
| 3 | บันทึก analytics event "order placed" | ระบบ analytics ของทีม marketing | ปลายทางและรูปแบบยังไม่ระบุ |

- Marketing คาดว่า**ปีหน้าจะมี post-order actions เพิ่มอีก** (SMS, fraud scoring, recommendations update) ดังนั้นจำนวนผู้ที่ต้องรู้ว่า "มีคำสั่งซื้อเกิดขึ้น" จะเพิ่มจาก 3 เป็นอย่างน้อย 6
- MailFast เป็นระบบภายนอก เราจึงควบคุม availability และ latency ของมันไม่ได้
- **ปัญหาที่ต้องตัดสินใจ:** service ที่รับคำสั่งซื้อควรแจ้ง post-order actions ทั้งหมดอย่างไร เพื่อให้เพิ่ม action ใหม่ได้ง่าย และไม่ให้ความล้มเหลวของ action ใดทำให้การสั่งซื้อล้มเหลวหรือช้าลง
- ไฟล์นี้เขียนจาก requirement เพียงอย่างเดียว (`context: none`) จึงไม่ได้อ่านโค้ดของระบบ ข้อมูลต่อไปนี้ Not evidenced in the provided files: ชื่อ service ที่รับคำสั่งซื้อ, protocol ระหว่าง services, ระบบมี message broker อยู่แล้วหรือไม่ และ `emailservice` ถูกเรียกอย่างไรในปัจจุบัน

## Decision drivers

เรียงจากสำคัญที่สุด:

1. **Modifiability / extensibility:** เพิ่ม post-order action ใหม่ (SMS, fraud scoring, recommendations) ได้โดยแก้ service ที่รับคำสั่งซื้อให้น้อยที่สุดหรือไม่ต้องแก้เลย
2. **Availability ของการสั่งซื้อ:** ถ้า MailFast, `loyaltyservice` หรือ analytics ล่ม การสั่งซื้อต้องยังสำเร็จ
3. **Performance:** latency ของการสั่งซื้อที่ผู้ใช้รู้สึกไม่ควรเพิ่มขึ้นตามจำนวน post-order actions
4. **Reliability:** event ต้องไม่หาย (อีเมลต้องถูกส่ง, ต้องให้แต้มครบ) และต้องไม่ให้แต้มซ้ำ (ต้องการ idempotency)
5. **Cost / operational complexity:** จำนวน infrastructure ใหม่ที่ทีมต้องดูแล
6. **Observability:** ติดตามได้ว่าคำสั่งซื้อใดทำ action ใดสำเร็จหรือล้มเหลวไปแล้ว

## Considered options

1. **A: Synchronous direct calls:** service ที่รับคำสั่งซื้อเรียก `emailservice` (ที่เรียก MailFast ต่อ), `loyaltyservice` และ analytics ทีละตัวโดยตรง ภายใน request เดียวกับการสั่งซื้อ
2. **B: Asynchronous publish/subscribe:** service ที่รับคำสั่งซื้อ publish event `OrderPlaced` เพียงครั้งเดียวไปยัง message broker แต่ละ action (`emailservice` → MailFast, `loyaltyservice`, analytics) เป็น consumer ของตัวเอง
3. **C: Post-order orchestrator service:** สร้าง service ใหม่ (เช่น `postorderservice`) ที่รับคำสั่งซื้อแบบ asynchronous หนึ่งครั้ง แล้วเป็นผู้เรียกทุก action ตามลำดับหรือแบบ workflow (มี retry และสถานะของแต่ละขั้น)

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| A: Synchronous direct calls | ง่ายที่สุด ไม่ต้องมี infrastructure ใหม่; รู้ผลทันทีว่าแต่ละ action สำเร็จหรือไม่; debug ง่ายเพราะเป็น call chain เดียว | ทุก action ใหม่ต้องแก้และ deploy service ที่รับคำสั่งซื้อ (modifiability ต่ำ); latency ของการสั่งซื้อเพิ่มตามจำนวน action และตาม latency ของ MailFast; MailFast ล่มอาจทำให้การสั่งซื้อล้มเหลวหรือค้าง (availability ต่ำ) เว้นแต่จะเขียน timeout/fallback เองทุกจุด; service ที่รับคำสั่งซื้อต้องรู้จักทุก downstream (coupling สูง) |
| B: Asynchronous pub/sub | เพิ่ม action ใหม่ได้โดยเพิ่ม consumer ใหม่ ไม่ต้องแก้ publisher (modifiability สูง); MailFast หรือ consumer ใดล่มไม่กระทบการสั่งซื้อ และ broker เก็บ event ไว้ retry ได้ (availability); latency ของการสั่งซื้อคงที่ = เวลา publish ครั้งเดียว (performance); consumer แต่ละตัว scale และ deploy แยกกันได้ | ต้องเพิ่มและดูแล message broker (cost, operational complexity); eventual consistency: อีเมลและแต้มมาช้ากว่าการสั่งซื้อเล็กน้อย; ได้ at-least-once delivery ทำให้ consumer ต้อง idempotent (โดยเฉพาะ loyalty points); ติดตาม flow ข้าม service ยากขึ้น ต้องมี tracing และ dead-letter queue; ถ้า publish หลัง commit คำสั่งซื้อแล้วล้มเหลว event อาจหาย (ต้องใช้ outbox หรือ retry) |
| C: Post-order orchestrator | การสั่งซื้อไม่ต้องรอ action (availability, performance ดีกว่า A); logic ลำดับขั้น, retry และสถานะอยู่ที่เดียว มองเห็นภาพรวมได้ง่าย (observability); รองรับขั้นตอนที่มีลำดับหรือเงื่อนไข เช่น fraud scoring ก่อนส่งอีเมลได้ดี | orchestrator ต้องแก้ทุกครั้งที่เพิ่ม action จึงกลายเป็นจุด coupling ใหม่ (modifiability ปานกลาง); เป็น service ใหม่ที่ต้องสร้างและดูแล และอาจต้องมี queue หรือ workflow engine อยู่ดี (cost); ถ้า orchestrator ล่ม action ทั้งหมดหยุดพร้อมกัน (single point of failure ของ post-order) |

## Decision

เลือก **Option B: Asynchronous publish/subscribe ด้วย event `OrderPlaced`** โดยมีรายละเอียดดังนี้

- service ที่รับคำสั่งซื้อ publish `OrderPlaced` **หนึ่งครั้ง** หลังบันทึกคำสั่งซื้อสำเร็จ แล้วตอบผู้ใช้ทันทีโดยไม่รอ post-order actions
- schema ของ event มี `order_id` (ใช้เป็น idempotency key), ข้อมูลลูกค้า/อีเมล, รายการสินค้า, ยอดรวม, timestamp และ `schema_version`
- consumers ในรอบนี้:
  - `emailservice`: เปลี่ยน logic จำลองเป็น adapter ที่เรียก **MailFast** โดยเก็บ MailFast ไว้หลัง `emailservice` เพื่อให้เปลี่ยน provider ได้ในอนาคตโดยไม่กระทบส่วนอื่น
  - `loyaltyservice` (สร้างใหม่): ให้แต้มแบบ idempotent ตาม `order_id`
  - analytics consumer: ส่ง event "order placed" ให้ระบบของทีม marketing
- ทุก consumer มี retry with backoff และ dead-letter queue
- เทคโนโลยีของ broker (เช่น RabbitMQ, Kafka, Google Cloud Pub/Sub) **ยังไม่ได้เลือก** และต้องตัดสินใจแยก (ดู Open questions)

**เหตุผล:** B ตอบ driver อันดับ 1 ได้ดีที่สุด เพราะ action ที่ marketing คาดไว้ปีหน้า (SMS, recommendations update) เพิ่มได้เป็น consumer ใหม่โดยไม่ต้องแก้ service ที่รับคำสั่งซื้อ และยังตอบ driver อันดับ 2–3 ด้วย เพราะแยกการสั่งซื้อออกจาก MailFast และ downstream อื่น A ไม่ผ่าน driver 1–3 ส่วน C ผ่าน driver 2–3 แต่ย้าย coupling ไปไว้ที่ orchestrator แทน ซึ่งขัดกับ driver 1 ค่าใช้จ่ายที่ต้องยอมรับคือ broker ใหม่และ eventual consistency (driver 5)

## Consequences

- **Positive:** เพิ่ม post-order action ใหม่ได้โดยไม่แก้ publisher เพียงแค่ subscribe `OrderPlaced` (quality attribute: modifiability)
- **Positive:** MailFast, `loyaltyservice` หรือ analytics ล่มแล้วการสั่งซื้อยังสำเร็จ event รอใน broker และถูก retry ภายหลัง (quality attribute: availability)
- **Positive:** latency ของการสั่งซื้อไม่เพิ่มตามจำนวน action หรือ latency ของ MailFast (quality attribute: performance)
- **Positive:** การเรียก MailFast อยู่ใน `emailservice` ที่เดียว ถ้าเปลี่ยน provider ก็แก้ที่เดียว (quality attribute: modifiability)
- **Positive:** consumer แต่ละตัว scale และ deploy แยกกันได้ (quality attribute: scalability, deployability)
- **Negative:** ต้องติดตั้ง ดูแล และ monitor message broker ใหม่ (quality attribute: cost, operability)
- **Negative:** อีเมลยืนยันและแต้มสะสมมาถึงช้ากว่าการสั่งซื้อ (eventual consistency) และหน้ายืนยันคำสั่งซื้อไม่สามารถยืนยันได้ว่าส่งอีเมลแล้ว (quality attribute: consistency, usability)
- **Negative:** debug flow ที่ข้ามหลาย consumer ยากกว่า call chain เดียว (quality attribute: observability, testability)
- **Risks and mitigations:**
  - *Event หายถ้าบันทึกคำสั่งซื้อสำเร็จแต่ publish ล้มเหลว* → ใช้ transactional outbox หรืออย่างน้อย retry การ publish แล้ว alert เมื่อล้มเหลว (reliability)
  - *ให้แต้มหรือส่งอีเมลซ้ำ เพราะ delivery เป็นแบบ at-least-once* → consumer ตรวจ `order_id` ก่อนทำงาน (idempotency) (reliability)
  - *MailFast ล่มนานหรือติด rate limit* → retry with backoff, circuit breaker ใน `emailservice` และ dead-letter queue พร้อม alert (availability)
  - *Schema ของ event เปลี่ยนแล้วทำให้ consumer พัง* → ใส่ `schema_version` และเปลี่ยนแบบ backward-compatible เท่านั้น (modifiability)
  - *Fraud scoring ในอนาคตอาจต้อง**บล็อก**คำสั่งซื้อก่อนยืนยัน* ซึ่ง pub/sub แบบหลังสั่งซื้อทำไม่ได้ → ต้องตัดสินใจใน ADR แยกเมื่อถึงเวลา (อาจเป็น synchronous check ก่อน publish) **[Inferred]**
  - *ข้อมูลส่วนบุคคล (อีเมลลูกค้า) ถูกส่งต่อให้ MailFast และ analytics* → ส่งให้ analytics เฉพาะ field ที่จำเป็น และตรวจสอบข้อกำหนดด้าน privacy (security)

## Validation

- **Load test:** วัด latency ของการสั่งซื้อ (เช่น p95) ก่อนและหลังเพิ่ม publish ต้องไม่เพิ่มเกินเป้าที่ตกลงกัน (เป้ายังไม่ได้กำหนด ดู Open questions)
- **Fault-injection test:** ปิด MailFast mock, `loyaltyservice` และ analytics consumer แล้วสั่งซื้อ ต้องสั่งซื้อสำเร็จ 100% และเมื่อเปิดกลับมา event ที่ค้างต้องถูกประมวลผลครบ
- **Idempotency test:** ส่ง `OrderPlaced` เดิมซ้ำ ต้องได้แต้มและอีเมลเพียงครั้งเดียว
- **Metrics ต่อเนื่อง:** consumer lag, จำนวน message ใน dead-letter queue, เวลาจากสั่งซื้อถึงส่งอีเมลสำเร็จ และอัตราความล้มเหลวของ MailFast
- **Modifiability check:** เมื่อเพิ่ม action แรกของปีหน้า (เช่น SMS) ต้องไม่ต้องแก้หรือ deploy service ที่รับคำสั่งซื้อ
- **Review date:** ทบทวน ADR นี้ก่อนเริ่มพัฒนา fraud scoring (ยังไม่มีวันที่ ดู Open questions)

## Open questions

ข้อมูลที่ requirement ไม่ได้ให้มา จึงไม่ได้สมมติขึ้นเอง:

1. **Service ใดรับและบันทึกคำสั่งซื้อ** และปัจจุบันเรียก `emailservice` อย่างไร (protocol, synchronous หรือไม่) Not evidenced in the provided files
2. **ระบบมี message broker อยู่แล้วหรือไม่** และทีมมีทักษะหรือข้อจำกัดด้านเทคโนโลยี/cloud ใด (ใช้เลือกระหว่าง RabbitMQ, Kafka, Pub/Sub ฯลฯ)
3. **เป้าหมายเชิงตัวเลข:** latency ของการสั่งซื้อ, ปริมาณคำสั่งซื้อ (orders/sec, peak) และเวลาสูงสุดที่ยอมรับได้จากสั่งซื้อถึงได้รับอีเมลหรือแต้ม
4. **MailFast:** API (REST/SMTP), SLA, rate limit, ค่าใช้จ่ายต่ออีเมล และรองรับ idempotency key หรือไม่
5. **Loyalty rules:** สูตรคิดแต้ม, ต้องหักแต้มคืนเมื่อยกเลิกหรือคืนสินค้าหรือไม่ และใครเป็นเจ้าของ `loyaltyservice`
6. **Analytics:** ปลายทางของทีม marketing คือระบบใด (external tool, data warehouse) และต้องการ field ใดบ้าง
7. **Fraud scoring:** ต้องบล็อกหรือยกเลิกคำสั่งซื้อได้หรือไม่ (ถ้าได้ ต้องอยู่ก่อน `OrderPlaced` ไม่ใช่หลัง)
8. **Privacy / compliance:** ข้อกำหนดเรื่องการส่งข้อมูลส่วนบุคคลให้ MailFast และ analytics
9. **Deciders และ review date** ของ ADR นี้
