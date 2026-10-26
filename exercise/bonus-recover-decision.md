**ภาษา:** ภาษาไทย | [English](bonus-recover-decision.en.md)

# โบนัส (ไม่บังคับ ทำที่บ้าน): กู้คืนการตัดสินใจที่ไม่ได้บันทึกไว้ (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T5 (Dhar et al. สร้างการตัดสินใจจากโค้ดและแผนภาพเพื่อลด *การสูญหายของความรู้* หรือ knowledge vaporization), AICH4
> **คำถาม:** เมื่อ AI อธิบายว่า*ทำไม*ระบบจึงถูกสร้างแบบนี้ คำอธิบายส่วนใดมาจากหลักฐาน และส่วนใดเป็นการเดาอย่างมั่นใจ

## ลงมือทำ

เริ่มแชตหรือ session ใหม่ แล้วส่งข้อความนี้ให้ agent

```
Follow prompts/bonus-recover-decision.md
decision: Why do all services communicate synchronously over gRPC, using one shared demo.proto?
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/src/checkoutservice/main.go, system/online-boutique/src/frontend/main.go
language: Thai
```

บรรทัด `language: Thai` ทำให้ ADR ที่กู้คืนเป็นภาษาไทย หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้

agent จะเขียน ADR ที่กู้คืนมา พร้อมตาราง **Claim check** ไว้ที่ [`outputs/bonus-recovered-adr.md`](outputs/bonus-recovered-adr.md)

## ตรวจสอบตาราง claim check

เลือกข้อกล่าวอ้าง**ห้าข้อ**จากตารางของ agent โดยเป็นข้อที่ agent ระบุว่า *Evidenced* (มีหลักฐาน) อย่างน้อยสองข้อ และ *Inferred* (อนุมานเอา) อย่างน้อยสองข้อ แล้วตรวจสอบแต่ละข้อด้วยตัวเอง

| # | ข้อกล่าวอ้าง (ย่อ) | AI ระบุว่า | นิสิตระบุว่า | เหตุผล (path ของไฟล์หรือการให้เหตุผล) |
|---|---|---|---|---|
| 1 | C7: หนึ่ง order ใช้ RPC แบบลำดับ 2N + 7 ครั้ง | Evidenced | Evidenced | นับจาก `src/checkoutservice/main.go`: `GetCart` (L325) 1 ครั้ง, loop ใน `prepOrderItems` (L343–348) เรียก `GetProduct` + `Convert` อย่างละ N ครั้ง = 2N, `GetQuote` (L314) 1, `Convert` ค่าส่ง (L302) 1, `Charge` (L370) 1, `ShipOrder` (L387) 1, `EmptyCart` (L333) 1, `SendOrderConfirmation` (L380) 1 รวม 2N + 7 ตรงกัน |
| 2 | C28: timeout ใน `mustConnGRPC` ไม่มีผล | Evidenced | Evidenced | `src/checkoutservice/main.go` L212 `_, cancel := context.WithTimeout(ctx, time.Second*3)` ทิ้ง context ที่มี timeout ไป และ `grpc.NewClient` (L214) ไม่ได้รับ context นั้น |
| 3 | C10: gRPC **ทุกเส้น**มี `otelgrpc` และใช้ W3C TraceContext | Evidenced | Evidenced บางส่วน (สรุปเกินหลักฐาน) | หลักฐานมีแค่ 2 จาก 9+ services: `checkoutservice/main.go` L139, L216, L135–137 และ `frontend/main.go` L109, L231 แต่คำว่า "ทุกเส้น" ครอบคลุม service ที่ไม่อยู่ในไฟล์หลักฐาน เมื่อตรวจ `src/cartservice/` แล้วไม่พบ OpenTelemetry เลย เส้นที่วิ่งเข้า `cartservice` จึงไม่มี tracing ฝั่ง server นอกจากนี้ exporter จะทำงานเฉพาะเมื่อ `ENABLE_TRACING == "1"` (`checkoutservice/main.go` L90, `frontend/main.go` L113) |
| 4 | C14: ต้องการ contract แบบ typed ที่รองรับหลายภาษา | Inferred | Inferred | ไฟล์หลักฐานทั้ง 3 ไฟล์เป็น Go/proto จึงไม่มีหลักฐานเรื่องหลายภาษา agent ไม่ได้อ้างเกินจริง แต่ใน `src/` มีหลายภาษาจริง ได้แก่ C# (`cartservice.csproj`), Node.js (`currencyservice/server.js`), Python (`emailservice/email_server.py`) และ Java (`adservice/build.gradle`) ส่วนที่ว่าเป็น "driver" ของการตัดสินใจยังเป็นการอนุมาน |
| 5 | C21: Option B และ D ถูกตัดเพราะไม่คุ้มสำหรับระบบตัวอย่างที่มีทีมเดียว | Inferred | Assumption | ไม่มีไฟล์ใดบอกว่าเคยพิจารณา B หรือ D และ agent ยอมรับเองว่า "จำนวนทีมเป็น Not evidenced" ข้อนี้จึงไม่มีหลักฐานให้อนุมาน ควรติดป้ายเป็น Assumption |

**นับ:** จากข้อกล่าวอ้างทั้งหมด agent ระบุว่า *Evidenced* กี่ข้อ **20** (C1–C11, C17, C19, C20, C24–C28, C30 โดย C8 ระบุว่า "เฉพาะ checkout") ตามที่นิสิตตรวจ **19** (C10 สรุปเกินหลักฐาน)

## สะท้อนคิด

1. agent เคยระบุการเดาว่าเป็น *Evidenced* หรือไม่ ยกตัวอย่าง หรืออธิบายว่าตรวจสอบอย่างไรว่าไม่มี

   เคย มี 1 ข้อคือ C10 ซึ่งสรุปจากหลักฐานของ 2 service ไปเป็น "ทุกเส้น" ส่วนข้ออื่นที่ระบุว่า Evidenced ตรวจเทียบกับเลขบรรทัดในไฟล์จริงแล้วตรงทุกข้อ

2. หากนักพัฒนาใหม่อ่านเฉพาะ ADR ที่กู้คืนมานี้ เขาอาจเข้าใจผิดในเรื่องใด

   เขาอาจเข้าใจว่าเหตุผลและ decision drivers ใน ADR เป็นสิ่งที่ผู้ออกแบบเดิมคิดไว้จริง ทั้งที่เกือบทั้งหมดเป็นการอนุมานของ AI และอาจเชื่อว่าทุก service ถูก trace ได้ครบเสมอ

3. AI กู้คืนอะไรจากโค้ดได้บ้าง และอะไรที่มีเพียงสถาปนิกผู้ออกแบบเดิมเท่านั้นที่รู้ ยกตัวอย่างหนึ่งตัวอย่างจาก ADR ของนิสิต

   AI กู้คืนได้ว่าระบบ*ทำอะไร*และทำอย่างไร เช่น รูปแบบการสื่อสาร ลำดับการเรียก และ config ที่มีอยู่ แต่กู้คืนไม่ได้ว่า*ทำไม*จึงเลือกแบบนี้ เช่น `PlaceOrderRequest` ข้าม field 4 (C4) AI เห็นว่าข้ามไป แต่เหตุผลที่ลบมีเพียงผู้ออกแบบเดิมเท่านั้นที่รู้
