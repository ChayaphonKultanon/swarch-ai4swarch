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
requirement:
After an order is placed, Online Boutique must:
1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
2. award **loyalty points** through a new `loyaltyservice` (to be built);
3. record an **"order placed" analytics event** for the marketing team.

Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: 
After an order is placed, Online Boutique must:
1. send the order confirmation email through **MailFast**, a new external email provider, which replaces the mock `emailservice` logic;
2. award **loyalty points** through a new `loyaltyservice` (to be built);
3. record an **"order placed" analytics event** for the marketing team.

Marketing expects **more post-order actions next year** (SMS, fraud scoring, a recommendations update).

language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | **ไม่ใช้เลย** เพราะไม่ได้รับ scenario (`context: none`) จึงย้ายเรื่อง latency, orders/s, SLA และ rate limit ของ MailFast ไปไว้ใน Open questions ข้อ 3–4 ส่วน Validation ระบุว่า "เป้ายังไม่ได้กำหนด" | **ใช้ครบ** ได้แก่ p95 ≤ 500 ms (ปัจจุบันประมาณ 300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / fail 0.5 % / 50 emails/s, loyalty ภายใน 1 นาที และ exactly once แล้วคำนวณต่อเป็นส่วนเกิน 150 emails/s ≈ 9,000 ต่อนาที (คำนวณถูก) |
| ทางเลือกที่พิจารณา (ระบุ) | 3 ทาง: (A) synchronous direct calls, (B) async pub/sub ผ่าน broker, (C) post-order orchestrator service **เลือก B แต่ยังไม่เลือก broker** (ยกตัวอย่าง RabbitMQ, Kafka, Pub/Sub) | 4 ทาง: (1) sync gRPC fan-out, (2) fire-and-forget goroutine, (3) RabbitMQ event, (4) Kafka event **เลือก RabbitMQ** โดยให้เหตุผลจากทักษะของทีมและข้อกำหนด self-hosted ไม่มีทางเลือกแบบ orchestrator |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | **Exactly once:** ใช้ `order_id` เป็น idempotency key ให้ consumer ตรวจก่อนทำงาน ไม่ได้ระบุว่าเก็บ key ไว้ที่ไหน **Rate limit:** ไม่รู้ตัวเลข จึงพูดแค่ "ติด rate limit → retry with backoff, circuit breaker, DLQ" ไม่มีกลไกคุมอัตราส่ง | **Exactly once:** dedup ด้วย `order_id` ผ่าน unique constraint ใน store ของ loyalty และแยก queue ของ loyalty ออกจาก email เพื่อไม่ให้ติด backlog **Rate limit:** token bucket ≤ 50/s รวมทุก replica แล้วให้ queue รับ backlog ช่วง peak พร้อม retry, backoff และ DLQ สำหรับ 0.5 % ที่ fail |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่พบตัวเลขที่แต่งขึ้น แต่มี **ข้อสมมติที่ไม่ได้ติดป้าย** คือ "publish หลังบันทึกคำสั่งซื้อสำเร็จ" ซึ่งสมมติว่ามี order store ทั้งที่โค้ดจริงของ `checkoutservice` ไม่มี database (`main.go` L230–280) | ข้อเท็จจริงจากโค้ดถูกทุกบรรทัดที่อ้าง (`main.go` L66–86, L115, L134–140, L233, L273–277, L379–384) แต่มี**ค่าที่ AI กำหนดเองโดยไม่ติดป้าย** ได้แก่ alert เมื่อ queue age > 30 s, RabbitMQ 3 nodes และ "p99 < 60 s" ซึ่งทำให้ข้อกำหนด "ภายใน 1 นาที" (ต้องทุก order) อ่อนลงเหลือ 99 % นอกจากนี้ภาษา `frontend` = Go และ Customer → frontend ผ่าน HTTPS ในแผนภาพก็ไม่มีอยู่ในไฟล์ context ส่วนตัวเลข "peak 10 นาที" มีบอกไว้แล้วว่าเป็นตัวอย่าง |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่ม **order store (PostgreSQL) และ transactional outbox** ใน `checkoutservice` แทนการ publish ตรงจาก `PlaceOrder` แบบ "ถ้าล้มเหลวให้ log แล้ว order สำเร็จ" และย้ายเรื่องนี้จาก Open questions มาเป็นส่วนหนึ่งของ Decision (เพิ่มตารางเปรียบเทียบ option A/B/C ของวิธี publish ด้วย) | ร่างของ AI ทำให้แต้มของ order หายถาวรเมื่อ publish ไม่สำเร็จ แต้มที่หาย (0 ครั้ง) ผิด "exactly once" เท่ากับแต้มซ้ำ ส่วนการให้ order fail เมื่อ publish ไม่ได้ก็ผิด "order must never fail" outbox เป็นทางเดียวที่ได้ทั้งสองข้อ เพราะเขียน order กับ event ใน transaction เดียวและเอา broker ออกจาก request path |
| 2 | เพิ่มหัวข้อ **การตีความ "exactly once"** ว่าเป็นผลลัพธ์ทางธุรกิจ (แต้มไม่หายและไม่ซ้ำ) ซึ่งประกอบจาก at-least-once delivery กับ idempotent consumer | broker ส่งแบบ exactly-once จริงไม่ได้ ร่างของ AI พูดถึงแค่ฝั่ง "ไม่ซ้ำ" (dedup) แต่ไม่ได้ป้องกันฝั่ง "ไม่หาย" การเขียนนิยามไว้ชัดทำให้อธิบายได้ว่าแต่ละกลไกรับผิดชอบครึ่งไหน |
| 3 | กำหนด **"ภายใน 1 นาที" ตามที่ scenario เขียน คือทุก order ไม่ใช่ p99** (แก้เกณฑ์ใน Validation จาก p99 < 60 s เป็นค่า max ≤ 60 s) และระบุข้อยกเว้นว่าในช่วงที่ broker หรือ `loyaltyservice` ล่ม แต้มจะช้าแต่ไม่หาย | AI เปลี่ยน requirement เป็น p99 เองโดยไม่มีเหตุผล ซึ่งที่ peak เท่ากับยอมให้ประมาณ 2 orders/s ช้าเกิน งบเวลาของการออกแบบ (relay poll 1 s + queue) ต่ำกว่า 60 s มากจึงไม่จำเป็นต้องผ่อน ส่วนช่วง component ล่ม การรักษา 1 นาทีมีทางเดียวคือให้ order fail จึงเขียนเป็น trade-off ที่ตั้งใจ และส่งให้ business ยืนยันใน Open questions |
| 4 | แก้ **วิธีคุม MailFast 50/s** เป็น token bucket ในตัว worker โดยใช้จำนวน replica คงที่ N (ไม่ใช้ HPA) และแต่ละ replica ≤ 50/N ต่อวินาที แทน "จำกัด prefetch และจำนวน consumer" ส่วน prefetch ใช้กำหนด concurrency เท่านั้น (≈ 75) | prefetch จำกัดแค่จำนวน message ที่ค้าง ack ไม่ได้จำกัดอัตรา อัตราจริงคือ in-flight ÷ latency ถ้า MailFast ตอบเร็วขึ้นจะเกิน 50/s ทันที ตาม Little's law ต้องมีงานค้างประมาณ 50/s × 1.5 s ≈ 75 จึงจะได้ throughput ถึง 50/s |
| 5 | เก็บ **RabbitMQ 3 nodes** ไว้ แต่เพิ่มเหตุผล และย้ายจาก mitigation ของ "ข้อมูลหาย" มาเป็นเรื่อง availability | AI ตั้งค่านี้เองโดยไม่มีที่มา quorum queues ใช้ Raft ที่ต้องมีเสียงข้างมาก 3 nodes คือจำนวนน้อยที่สุดที่ทนได้ 1 node ล่ม เมื่อมี outbox แล้ว broker ล่มไม่ทำให้ข้อมูลหาย 3 nodes จึงมีไว้รักษางบ 1 นาที |
| 6 | เก็บ **alert 30 s** ไว้ในฐานะค่าเริ่มต้นที่ปรับได้ พร้อมเหตุผล และเพิ่ม alert อายุแถวที่ยังไม่ publish ใน outbox | AI ตั้งค่านี้เองโดยไม่มีที่มา 30 s คือครึ่งหนึ่งของงบ 60 s เป็นสัญญาณเตือนล่วงหน้าก่อนละเมิด SLO จริง และเมื่อมี outbox ต้อง monitor ทั้งฝั่ง outbox และฝั่ง queue จึงจะเห็นปัญหาของ relay หรือ broker |
| 7 | เพิ่ม Consequence ว่า **มี stateful component ใหม่สองตัว** และ **PostgreSQL กลายเป็น critical path** ของ checkout | เป็นราคาที่ต้องจ่ายของ outbox ต้องเขียนไว้ตรง ๆ และอธิบายว่าไม่ขัด scenario เพราะ availability constraint ครอบคลุมแค่ email, loyalty และ analytics ส่วน order store คือ state หลักของ checkout เอง |
| 8 | แก้ข้อเสียของ **Kafka** จาก "ขัดกับเงื่อนไข" เป็น "retry/DLQ ไม่มีในตัว ต้องสร้างเพิ่มเอง" | ร่างของ AI กล่าวเกินจริง Kafka ทำ retry และ DLQ ได้ เพียงต้องเขียนเพิ่ม การเขียนให้ตรงทำให้การปฏิเสธ Kafka ยืนอยู่บนเหตุผลที่ถูกต้อง (ประสบการณ์ทีมและภาระ operate) |
| 9 | เพิ่ม **Validation**: fault injection ที่ปิด RabbitMQ ทั้ง cluster และ duplicate test ที่ kill relay หลัง publish แต่ก่อน mark | ร่างของ AI ไม่ได้ทดสอบกรณีที่ outbox มีไว้แก้ (broker ล่ม) และกรณี publish ซ้ำที่ outbox สร้างขึ้นเอง ถ้าไม่ทดสอบสองกรณีนี้จะพิสูจน์ "ไม่หายและไม่ซ้ำ" ไม่ได้ |
| 10 | ปรับ **Open questions**: ลบเรื่องที่ตัดสินแล้ว (outbox, ผู้อนุมัติ) และเพิ่มเรื่องประสบการณ์ PostgreSQL ของทีม, รายละเอียด rate limit ของ MailFast, latency ของ transaction และกรณีตัดเงินแล้วเขียน order ไม่สำเร็จ | Open questions ต้องเหลือเฉพาะสิ่งที่ยังไม่รู้จริง เรื่องที่สถาปนิกตัดสินแล้วต้องอยู่ใน Decision ส่วนคำถามใหม่เกิดจากการเพิ่ม PostgreSQL และ outbox |
