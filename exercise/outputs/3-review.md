Reviewer: Claude Code (model: Claude Opus 5.5, `claude-opus-5-5`)

# Review: ADR-0002 `OrderPlaced` ผ่าน transactional outbox + RabbitMQ

ADR ที่ review: `docs/adr/0002-publish-order-placed-event-via-outbox-and-rabbitmq.md` ตัวเลขทั้งหมดมาจาก `docs/scenario.md`

## 1. Quality-attribute scenarios

| # | Source · Stimulus | Environment | Response | Response measure |
|---|---|---|---|---|
| S1 Performance | ลูกค้าส่ง `PlaceOrder` 200 ครั้ง/s | peak 11.11 ทุก component ปกติ | commit `orders` + `outbox` ใน transaction เดียวแล้วตอบ client | p95 ≤ 500 ms (headroom ~200 ms จาก ~300 ms) |
| S2 Availability | RabbitMQ ทั้ง cluster ล่ม 5 นาที | peak 200 orders/s | order สำเร็จ แถวสะสมใน outbox (~60,000 แถว) แล้วระบายเมื่อ broker กลับมา | order fail เพราะ broker = 0 %, p95 ยัง ≤ 500 ms, event หาย = 0 |
| S3 Correctness | relay crash หลัง publish ก่อน mark `published_at` | สภาวะปกติ 200 orders/s | `loyaltyservice` ชน unique constraint บน `order_id` แล้ว ack ทิ้ง | แต้มต่อ order = 1 พอดี, เวลาถึงแต้ม max ≤ 60 s |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Risk | Relay ใช้ `SELECT ... FOR UPDATE SKIP LOCKED` แล้ว publish พร้อม publisher confirms ก่อน mark (ADR บรรทัด 82) ไม่ได้ระบุว่า lock/connection ถูกถือไว้ระหว่างรอ confirm หรือไม่ และใช้ connection pool ร่วมกับ `PlaceOrder` หรือเปล่า | ถ้า broker ล่มหรือช้า transaction ของ relay ค้าง pool เต็ม แล้ว `PlaceOrder` fail **เพราะ broker** ซึ่งผิด driver 1 และทำให้ S2 ไม่ผ่าน |
| Risk | PostgreSQL เข้ามาอยู่ใน critical path แบบอนุกรม (บรรทัด 149) แต่ประสบการณ์ PostgreSQL ของทีมยังไม่รู้ (บรรทัด 176) | Kafka ถูกตัดเพราะทีมไม่มีประสบการณ์ แต่ PostgreSQL HA ที่ไม่มีหลักฐานเรื่องทักษะกลับถูกรับ เหตุผลจึงไม่สม่ำเสมอ และ availability ของ checkout ลดลงเทียบกับวันนี้ |
| Risk | Risks ข้อ "consumer ทุกตัวต้อง idempotent" (บรรทัด 154) แต่มีกลไก dedup แค่ฝั่ง loyalty ส่วน `emailservice` ไม่มี store หรือ idempotency key (บรรทัด 93–97, 174) | redeliver หรือ relay publish ซ้ำ ลูกค้าจะได้อีเมลยืนยันซ้ำ ซึ่งเป็นปัญหาที่ลูกค้าเห็นได้ |
| Sensitivity point | Replication mode ของ PostgreSQL HA (sync หรือ async) ซึ่ง ADR ไม่ได้ตัดสิน | sync เพิ่ม round-trip ใน commit และกิน headroom 200 ms (S1) ส่วน async ทำให้แถว outbox ที่ commit แล้วหายตอน failover ได้ แต้มจึงหาย (S3) |
| Sensitivity point | อัตรารวมไปยัง MailFast = N × (50/N) โดยสมมติว่า N คงที่ (บรรทัด 95) | ระหว่าง rolling update ที่มี `maxSurge` หรือเมื่อมีคน scale ด้วยมือ ค่า N จริงจะเกินค่าที่ตั้งไว้ และ retry ก็ใช้ quota เดียวกัน อัตราจึงอาจเกิน 50/s |
| Sensitivity point | Polling interval, batch size และ retention ของ outbox | ที่ 200/s มีแถวเข้าและถูกลบ ~720,000 แถว/ชม. ถ้า table bloat หรือ vacuum ไม่ทัน latency ของ insert ใน `PlaceOrder` จะเพิ่ม และ batch size เป็นตัวกำหนดว่า backlog 60,000 แถวจาก S2 จะระบายเร็วแค่ไหน |
| Trade-off point | Outbox: durability และ availability ต่อ broker ↑ แลกกับ latency, operability (stateful 2 ตัว) และ dependency ใหม่ใน critical path ↓ | เป็นแกนของการตัดสินใจ ADR ระบุไว้ตรงไปตรงมาแล้ว (บรรทัด 148–149) |
| Trade-off point | ให้ availability ชนะ "ภายใน 1 นาที" ตอนที่ broker หรือ loyalty ล่ม (บรรทัด 39) | การตัดสินใจถูกทาง แต่เป็นการผ่อน requirement ที่ business ยังไม่ได้อนุมัติ |
| Non-risk | Queue แยกต่อ consumer + quorum queues 3 nodes (บรรทัด 88–89) | backlog ของ MailFast ไม่บล็อก loyalty และเหตุผลเรื่อง Raft majority ถูกต้อง |
| Non-risk | Loyalty dedup ด้วย unique `order_id` และ ack หลัง commit (บรรทัด 98) | ได้ exactly-once ทางธุรกิจจริงจาก at-least-once + idempotent consumer |
| Non-risk | คุม rate ด้วย token bucket ไม่ใช่ prefetch (บรรทัด 94–96) | ใช้ Little's law ถูกต้อง และไม่ผูกอัตรากับ latency ของ MailFast |

## 3. Missing or weak

- **Options ที่ไม่ได้พิจารณา:**
  - CDC (เช่น Debezium อ่าน WAL) แทน polling relay: ตัด polling load และไม่มี lock บน critical DB แต่เพิ่ม component
  - Outbox-only table ไม่ต้องมี `orders` เต็ม: R-12 ไม่ได้ต้องการ order store จึงลด scope, schema และ PII ลงได้
  - เขียน order สถานะ `PENDING` **ก่อน** `chargeCard`: แก้ปัญหา "ตัดเงินแล้วไม่มี order" ที่ ADR นี้ทำให้มีจุด fail เพิ่ม
  - RabbitMQ Streams สำหรับ analytics replay: ตอบ open question บรรทัด 178 ได้โดยไม่ต้องเพิ่ม Kafka
- **Consequences ที่ไม่ได้เขียน:** relay แย่ง CPU กับ request ใน pod ของ checkout และ HPA ของ checkout เปลี่ยนจำนวน relay ไปด้วย, checkout กลายเป็น stateful (migration, backup), อีเมลซ้ำ, เวลาระบาย outbox หลัง broker กลับมา
- **Claims ที่ไม่มีหลักฐานหรือน่าจะผิด:**
  - "ตัดเงินแล้วเขียน order ไม่สำเร็จเป็นปัญหาเดิม อยู่นอกขอบเขต" (บรรทัด 156): ผิด เพราะจุด fail นี้เกิดใหม่จาก ADR นี้เอง จึงอยู่ในขอบเขต
  - "PostgreSQL ใน critical path ไม่ขัดกับ scenario" (บรรทัด 149): ถูกตามตัวอักษร แต่ไม่ได้ยอมรับว่า availability โดยรวมของ checkout ลดลง
  - "p95 ยังอยู่ใน headroom" (บรรทัด 143): call ไป `emailservice` ที่ตัดออกเป็น mock จึงน่าจะประหยัดได้น้อย ขณะที่ HA commit มีต้นทุน ทั้งหมดนี้ยังไม่ได้วัด (ติด label [Inferred] ไว้แล้ว)
  - **Status: Accepted** ทั้งที่ข้อยกเว้น 1 นาทีกับ latency ของ PostgreSQL ยังรอ business และการวัด ควรเป็น `Proposed`

## 4. Verdict: **Accept with changes**

1. **แยก relay ออกจาก request path:** ใช้ connection pool แยก, claim แถวแล้ว publish นอก transaction ที่ถือ lock โดยมี timeout และเพิ่มใน Validation ว่าตอน broker ล่มที่ 200 orders/s ค่า p95 ยัง ≤ 500 ms
2. **ตัดสิน PostgreSQL HA และ replication mode** (แนะนำ sync replica) วัด commit latency เทียบกับ headroom 200 ms ยืนยันทักษะทีม และเขียน order `PENDING` ก่อน `chargeCard`
3. **ทำ `emailservice` ให้ idempotent** (dedup ด้วย `order_id` หรือ idempotency key ของ MailFast) ล็อก rate ไม่ให้ขึ้นกับ N (`maxSurge: 0` หรือ limiter กลาง) และเปลี่ยน status เป็น `Proposed` จนกว่า business จะอนุมัติข้อยกเว้น
