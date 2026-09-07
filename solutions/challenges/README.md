# เฉลยโจทย์ประจำวัน

> ⚠️ อ่านก่อนลอง = เสียโอกาสเรียนรู้ · ดูวิธีใช้ที่ [../README.md](../README.md)

| โจทย์ | ไฟล์ | คำสั่ง |
|---|---|---|
| [1 · Thai Token Audit](../../instructions/day1/challenge1-thai-token-audit.md) | `challenge1_token_audit.py` | `uv run solutions/challenges/challenge1_token_audit.py` |
| [2 · Schema Under Pressure](../../instructions/day1/challenge2-schema-under-pressure.md) | `challenge2_robust_extractor.py` | `uv run solutions/challenges/challenge2_robust_extractor.py` |
| [3 · Tool Description Battle](../../instructions/day2/challenge3-tool-description-battle.md) | `challenge3_descriptions.json` | ดูวิธีใช้ด้านล่าง |
| [4 · Topic Shift Survival](../../instructions/day2/challenge4-topic-shift-survival.md) | `challenge4_topic_shift.py` | `uv run solutions/challenges/challenge4_topic_shift.py` |
| 5 · Guardrail Red-team | `tests/test_guardrails.py` | `uv run pytest tests/test_guardrails.py` |
| 6 · Cross-Service Diagnosis | `make eval` | ดู `eval/results/latest.json` |

---

## โจทย์ที่ 1 — สิ่งที่ควรได้จากตัวเลข

สคริปต์ตอบ 3 คำถาม:

1. **ภาษาไทยแพงกว่าอังกฤษกี่เท่า** ที่จำนวนอักขระเท่ากัน
2. **tiktoken ผิดกี่เปอร์เซ็นต์** เมื่อเอามาใช้กับ Gemma
3. **ต้นทุนจริงเมื่อขึ้น production** 2,600 อุปกรณ์

ข้อ 2 คือข้อสรุปที่สำคัญที่สุด — ถ้าใช้ตัวเลขจาก tokenizer ผิดรุ่นไปวางแผนงบประมาณหรือกำหนดขนาด chunk จะผิดตั้งแต่ต้นโดยไม่มีอะไรเตือน

สคริปต์ใช้ `setseed(0.42)` เพื่อให้สุ่มตัวอย่างได้ผลเดิมทุกครั้ง — ตัวเลขในรายงานจึงสร้างซ้ำได้

---

## โจทย์ที่ 2 — 4 การป้องกัน

| ป้องกัน | แก้ปัญหาของ | วิธี |
|---|---|---|
| ตัดข้อความยาว | ใบที่ 2 (40 ข้อความ) | เก็บ**หัวและท้าย** ทิ้งตรงกลาง |
| Delimiter + system prompt | ใบที่ 4 (injection) | แยก "คำสั่ง" ออกจาก "ข้อมูล" ให้ชัด |
| `confidence` ต่ำเมื่อข้อมูลน้อย | ใบที่ 1, 3 | บังคับใน field description |
| Circuit breaker | ปัญหาเชิงระบบ | ล้มเหลวติดกัน 5 ครั้ง = หยุด |

**ทำไมตัดตรงกลาง ไม่ใช่ตัดท้าย**: ตรงกลางคือจุดที่โมเดลใช้ข้อมูลได้แย่ที่สุดอยู่แล้ว (Module 2) และข้อความแรกๆ บอกอาการ ส่วนข้อความท้ายๆ บอกผลการแก้ไข — ซึ่งเป็นสองสิ่งที่กำลังสกัดพอดี

**การตรวจจับ injection ไม่ใช่การป้องกัน** — delimiter ต่างหากที่ป้องกัน การตรวจจับมีไว้เพื่อ log และลด confidence เพราะ ticket ที่พยายามหลอกระบบเป็น ticket ที่คนควรดู

---

## โจทย์ที่ 3 — วิธีใช้เฉลย

1. เปิด `challenge3_descriptions.json`
2. คัดลอกส่วน `descriptions_under_test` ไปแทนที่ใน `data/challenge_fixtures/tool_selection_cases.json`
3. รัน `uv run pytest tests/test_tool_selection.py`

คะแนนควรขึ้นจาก ~5/12 เป็น 11-12/12

**อ่านส่วน `why_it_works` และ `cases_still_hard` ด้วย** — ข้อ TS-12 ยังพลาดได้แม้ description ดีแล้ว เพราะเป็นสัญญาณว่า tool ออกแบบทับซ้อนกันตั้งแต่แรก ซึ่งเป็นบทเรียนที่ลึกกว่าการแก้ prose

---

## โจทย์ที่ 4 — กราฟคือคำตอบ

สคริปต์พิมพ์กราฟแท่งของ `context_tokens` ทั้ง 10 turn และตรวจ 3 จุด:

| turn | ต้องเกิดอะไร |
|---|---|
| **5** | นอกขอบเขต · `tool_calls == 0` · **context ไม่ถูกล้าง** |
| **6** | เปลี่ยนเรื่อง · context **ลดลงอย่างน้อย 40%** |
| **9** | ย้อนเรื่องเดิม · ตอบได้จากสรุป · เรียก tool ไม่เกิน 1 ครั้ง |

ถ้ากราฟเป็นเส้นที่โตขึ้นตลอด แปลว่ากลไกความจำยังไม่ทำงาน ไม่ว่าคำตอบแต่ละ turn จะดูดีแค่ไหน

ผลถูกบันทึกไว้ที่ `challenge4_result.json` เอาไปวาดกราฟต่อได้

---

## โจทย์ที่ 5 และ 6 — ทำไมไม่มีไฟล์เฉลย

**โจทย์ที่ 5** เป็นการทดลองเชิงปฏิบัติ (แดง/น้ำเงิน) ที่ไม่มี "คำตอบเดียว" — สิ่งที่ใกล้เคียงเฉลยที่สุดคือ `tests/test_guardrails.py` ซึ่งกำหนดว่าอะไรบ้างที่ต้องถูกบล็อก

> การทดลองที่สำคัญที่สุดของโจทย์นี้คือ **ปิด guardrail ทีละชั้นแล้วทดสอบซ้ำ**
> คำตอบที่ต้องค้นพบเอง: ปิดชั้นบนได้ ระบบยังปลอดภัย แต่พอเปลี่ยนบัญชีฐานข้อมูล ทุกอย่างพังทันที

**โจทย์ที่ 6** วัดด้วย `make eval` ซึ่งรันชุดคำถาม L3 ทั้งหมดและรายงานว่าข้อไหนผ่าน — ตรงกว่าการมีไฟล์เฉลย
