# Workshop 1 · โมดูลบังคับ JSON พร้อม Auto-retry

**14:45 – 16:00** (75 นาที)

---

## โจทย์

สร้างคลาสที่รับ input จากผู้ใช้ ส่งไปประมวลผลกับ LLM โดย **บังคับโครงสร้าง output ที่แน่นอน** และถ้าได้ JSON ที่ไม่ถูกต้อง ให้ **แก้ไขและลองใหม่อัตโนมัติ**

ผลงานชิ้นนี้จะถูกใช้ต่อในวันที่ 2 (สร้าง plan) และวันที่ 3 (structured output ของ MCP tool) จึงควรเขียนให้ใช้ซ้ำได้

---

## บริบทงานจริง

`ticket_messages` ในระบบเป็นข้อความอิสระที่ลูกค้าและเจ้าหน้าที่พิมพ์กันเอง ไทยปนอังกฤษ ไม่มีโครงสร้าง

ต้องแปลงให้เป็นข้อมูลที่ระบบใช้ต่อได้: สร้างไฟล์ `test_parser.py`

```python
import asyncio
import sys
sys.path.insert(0, 'apps/agent-api')

from pydantic import BaseModel, Field
from agent import llm

# 1. สร้าง Schema กำหนดโครงสร้าง Output ด้วย Pydantic
class TicketSummary(BaseModel):
    category: str = Field(description="ประเภทของปัญหา เช่น intermittent, offline, latency")
    severity: str = Field(description="ระดับความรุนแรง เช่น high, medium, low")
    affected_device: str = Field(description="ชื่ออุปกรณ์ที่ได้รับผลกระทบ (ถ้ามี)")
    affected_site: str = Field(description="ชื่อสาขาหรือไซต์ที่ได้รับผลกระทบ")
    summary_th: str = Field(description="สรุปปัญหาที่เกิดขึ้นเป็นภาษาไทยสั้นๆ")
    customer_impact: str = Field(description="ผลกระทบต่อการใช้งานของลูกค้า")
    confidence: float = Field(description="ความมั่นใจในการสรุปข้อมูล (0.0 ถึง 1.0)")

# 2. สร้าง Class สำหรับนำ Input ไปประมวลผล
class TicketParser:
    async def parse(self, text: str) -> TicketSummary:
        messages = [
            {
                "role": "system",
                "content": (
                    "คุณคือผู้เชี่ยวชาญด้าน IT Support จงสกัดข้อมูลจากข้อความแจ้งปัญหาของลูกค้า "
                    "และสรุปออกมาเป็นโครงสร้าง JSON ตามที่กำหนด"
                )
            },
            {
                "role": "user",
                "content": text
            }
        ]
        
        # ใช้ complete_structured เพื่อบังคับ Output และทำ Auto-correction
        result = await llm.complete_structured(messages, TicketSummary)
        return result

# 3. ฟังก์ชันทดสอบรันการทำงาน
async def main():
    parser = TicketParser()
    
    # จำลองข้อความดิบที่ลูกค้ารายงานมา (ไทยปนอังกฤษ ไม่มีโครงสร้าง)
    raw_text = "ลูกค้าสาขา NBI โทรมาโวยวายว่าเน็ตหลุดเป็นช่วงๆ ตั้งแต่เช้า ใช้งาน video conference ไม่ต่อเนื่องเลย แจ้งให้เช็คเร้าเตอร์ LPE-NBI-11 ด่วนๆ"
    
    print("กำลังประมวลผล...")
    structured_data = await parser.parse(raw_text)
    
    # พิมพ์ผลลัพธ์ออกมาดูในรูปแบบ JSON
    print("\n✅ ผลลัพธ์ที่ได้:")
    print(structured_data.model_dump_json(indent=2, ensure_ascii=False))

if __name__ == "__main__":
    asyncio.run(main())
```

```python
uv run test_parser.py
```

---

## สิ่งที่ต้องสร้าง

### 1. Schema

```python
from enum import Enum
from pydantic import BaseModel, Field

class Category(str, Enum):
    LINK_DOWN = "link_down"
    INTERMITTENT = "intermittent"
    SLOW = "slow"
    CONFIG = "config"
    MAINTENANCE = "maintenance"
    INQUIRY = "inquiry"

class Severity(str, Enum):
    LOW = "low"; MEDIUM = "medium"; HIGH = "high"; CRITICAL = "critical"

class TicketExtraction(BaseModel):
    category: Category
    severity: Severity
    affected_device: str | None = Field(None, description="รหัสอุปกรณ์ เช่น LPE-NBI-11 ถ้าไม่มีให้เป็น null")
    affected_site: str | None = Field(None, description="BKK หรือ NBI เท่านั้น")
    summary_th: str = Field(description="สรุปภาษาไทยไม่เกิน 2 ประโยค")
    customer_impact: str = Field(description="ผลกระทบต่อการใช้งานของลูกค้า")
    confidence: float = Field(ge=0.0, le=1.0)
```

### 2. คลาส `StructuredExtractor`

```python
class StructuredExtractor:
    def __init__(self, schema, model=None, max_retries=3, temperature=0.0): ...

    async def extract(self, text: str) -> ExtractionResult:
        """คืนผลลัพธ์พร้อมข้อมูลว่า retry ไปกี่ครั้งและเพราะอะไร"""
```

**ต้องมี**

| ความสามารถ | รายละเอียด |
|---|---|
| บังคับ schema | ใช้ guided decoding ถ้า gateway รองรับ |
| Auto-retry | ส่ง validation error กลับไปให้โมเดลแก้ |
| ทำความสะอาด output | ตัด ` ```json ` และคำนำที่โมเดลชอบใส่ |
| บันทึกความพยายาม | เก็บว่าแต่ละครั้งผิดอะไร |
| Fallback | เมื่อ retry ครบแล้วยังไม่ผ่าน **ห้าม crash** |
| นับ token | รายงาน token ที่ใช้ทั้งหมดรวมทุก retry |

### 3. ผลลัพธ์ที่คืน

```python
class ExtractionResult(BaseModel):
    ok: bool
    data: TicketExtraction | None
    attempts: int
    errors: list[str]
    total_tokens: int
    latency_ms: int
    fallback_used: bool = False
```

---

## ทดสอบกับข้อมูลจริง

```sql
SELECT t.ticket_id,
       string_agg(m.author_role || ': ' || m.message, E'\n' ORDER BY m.created_at) AS conversation
FROM tickets t JOIN ticket_messages m ON m.ticket_id = t.ticket_id
GROUP BY t.ticket_id LIMIT 20;
```

## สร้างไฟล์ extractor.py

```python
import asyncio
import os
import sys
from enum import Enum
from pathlib import Path
from typing import Any
from pydantic import BaseModel, Field
import psycopg

# เพิ่ม Path ไปยังโฟลเดอร์ agent-api
sys.path.insert(0, str(Path(__file__).resolve().parent / "apps" / "agent-api"))
from agent import llm

PG_DSN = os.getenv("PG_ADMIN_DSN", "postgresql://mpls:mpls_dev_password@localhost:5432/mplsdb")

# ==========================================
# 1. Schema กำหนดโครงสร้างข้อมูล
# ==========================================
class Category(str, Enum):
    LINK_DOWN = "link_down"
    INTERMITTENT = "intermittent"
    SLOW = "slow"
    CONFIG = "config"
    MAINTENANCE = "maintenance"
    INQUIRY = "inquiry"

class Severity(str, Enum):
    LOW = "low"
    MEDIUM = "medium"
    HIGH = "high"
    CRITICAL = "critical"

class TicketExtraction(BaseModel):
    category: Category
    severity: Severity
    affected_device: str | None = Field(None, description="รหัสอุปกรณ์ เช่น LPE-NBI-11 ถ้าไม่มีให้เป็น null")
    affected_site: str | None = Field(None, description="BKK หรือ NBI เท่านั้น")
    summary_th: str = Field(description="สรุปภาษาไทยไม่เกิน 2 ประโยค")
    customer_impact: str = Field(description="ผลกระทบต่อการใช้งานของลูกค้า")
    confidence: float = Field(ge=0.0, le=1.0)

# โครงสร้างผลลัพธ์ตามโจทย์เป๊ะ
class ExtractionResult(BaseModel):
    ok: bool
    data: TicketExtraction | None = None
    attempts: int
    errors: list[str]
    total_tokens: int
    latency_ms: int
    fallback_used: bool = False


# ==========================================
# 2. คลาส StructuredExtractor 
# ==========================================
class StructuredExtractor:
    def __init__(self, schema, model=None, max_retries=3, temperature=0.0):
        self.schema = schema
        self.model = model
        self.max_retries = max_retries
        self.temperature = temperature

    async def extract(self, text: str) -> ExtractionResult:
        stats = llm.LLMStats()
        json_schema = self.schema.model_json_schema()
        
        messages = [
            {
                "role": "system",
                "content": (
                    "คุณคือ AI สกัดข้อมูล Ticket เน็ตเวิร์ก จงแปลงบทสนทนานี้เป็น JSON ตาม Schema เท่านั้น "
                    "ห้ามใส่ข้อความอื่นนอกเหนือจาก JSON\n"
                    f"Schema: {json_schema}"
                )
            },
            {"role": "user", "content": text}
        ]

        errors_log = []
        fallback_used = False

        for attempt in range(1, self.max_retries + 1):
            try:
                # เรียกใช้ LLM ผ่านโมดูลกลางพร้อมเก็บ Stats (Token & Latency)
                raw_response = await llm.complete(
                    messages=messages,
                    stats=stats,
                    model=self.model,
                    temperature=self.temperature
                )
                
                # ทำความสะอาด Markdown Fence ถ้ามี
                clean_raw = raw_response.strip()
                if clean_raw.startswith("```"):
                    clean_raw = clean_raw.split("```")[1]
                    if clean_raw.startswith("json"):
                        clean_raw = clean_raw[4:]
                clean_raw = clean_raw.strip()

                # ตรวจสอบความถูกต้องด้วย Pydantic Schema
                parsed_data = self.schema.model_validate_json(clean_raw)
                
                stats_dict = stats.as_dict()
                return ExtractionResult(
                    ok=True,
                    data=parsed_data,
                    attempts=attempt,
                    errors=errors_log,
                    total_tokens=stats_dict["total_tokens"],
                    latency_ms=stats_dict["latency_ms"],
                    fallback_used=fallback_used
                )

            except Exception as exc:
                error_msg = str(exc)
                errors_log.append(error_msg)
                
                # ส่งประวัติที่ผิดพลาดกลับไปให้ LLM แก้ตัวเองในรอบถัดไป
                messages.append({"role": "assistant", "content": raw_response[:1000] if 'raw_response' in locals() else ""})
                messages.append({
                    "role": "user",
                    "content": f"JSON ไม่ผ่านการตรวจสอบ Validation Error: {error_msg}\nกรุณาแก้ไขและส่งเฉพาะ JSON ที่ถูกต้องเท่านั้น"
                })

        # ถ้ายอมแพ้หลังจากลองจนครบจำนวนครั้ง
        stats_dict = stats.as_dict()
        return ExtractionResult(
            ok=False,
            data=None,
            attempts=self.max_retries,
            errors=errors_log,
            total_tokens=stats_dict["total_tokens"],
            latency_ms=stats_dict["latency_ms"],
            fallback_used=fallback_used
        )


# ==========================================
# 3. ทดสอบดึงข้อมูลจริงจาก PostgreSQL ตาม SQL ในโจทย์
# ==========================================
async def main():
    print("กำลังเชื่อมต่อฐานข้อมูลเพื่อดึง Ticket จริง...")
    
    with psycopg.connect(PG_DSN) as conn:
        cur = conn.cursor()
        # ใช้ SQL Query ตามโจทย์เป๊ะเพื่อดึงบทสนทนาของแต่ละ Ticket
        rows = cur.execute("""
            SELECT t.ticket_id, 
                   string_agg(m.author_role || ': ' || m.message, E'\n' ORDER BY m.created_at) AS conversation
            FROM tickets t JOIN ticket_messages m ON m.ticket_id = t.ticket_id
            GROUP BY t.ticket_id LIMIT 3;
        """).fetchall()

    extractor = StructuredExtractor(schema=TicketExtraction)

    for ticket_id, conversation in rows:
        print(f"\n----------------------------------------")
        print(f"📌 กำลังประมวลผล Ticket ID: {ticket_id}")
        
        result = await extractor.extract(conversation)
        
        print(f"สถานะ (OK): {result.ok}")
        print(f"จำนวนรอบที่ใช้ (Attempts): {result.attempts}")
        print(f"เวลาที่ใช้ (Latency): {result.latency_ms} ms | Tokens: {result.total_tokens}")
        if result.errors:
            print(f"ข้อผิดพลาดที่พบ: {result.errors}")
        
        if result.ok and result.data:
            print("ผลลัพธ์ที่สกัดได้:")
            print(result.data.model_dump_json(indent=2, ensure_ascii=False))

if __name__ == "__main__":
    asyncio.run(main())
```

```python
uv run extractor.py
```

---

## เกณฑ์ผ่าน

- [ ] ประมวลผล ticket ทั้ง 20 ใบได้ JSON ที่ผ่าน validation ครบ
- [ ] ระบบไม่ crash แม้แต่ครั้งเดียว
- [ ] มี log ว่า retry กี่ครั้ง และแต่ละครั้งผิดเพราะอะไร
- [ ] รายงาน token ที่ใช้รวม
- [ ] มี fallback เมื่อ retry ครบ

---

## โบนัส

1. **วัดผลของ guided decoding** — รันด้วย `LLM_GUIDED_DECODING=true` และ `false` อย่างละ 20 ครั้ง เทียบอัตราสำเร็จครั้งแรกและ token ที่ใช้
2. **เทียบโมเดล** — `gemma3:4b` กับ `gemma3:27b` ตัวเล็กพลาดบ่อยกว่ากี่เท่า
3. **ตรวจความสมเหตุสมผลข้ามฟิลด์** — ถ้า `affected_device = "LPE-NBI-11"` แต่ `affected_site = "BKK"` ต้องจับได้ (ใช้ Pydantic `model_validator`)

---

<details>
<summary>Hint</summary>

- ดูโครงที่ `apps/agent-api/agent/llm.py` → `complete_structured()` แต่**อย่าลอกทั้งดุ้น** เขียนเองแล้วค่อยเทียบ
- ตัด markdown fence: `if raw.startswith("```"): raw = raw.split("```")[1]` แล้วตัด `json` ที่ขึ้นต้น
- Pydantic ให้ error ที่อ่านรู้เรื่องอยู่แล้ว ส่ง `str(exc)` กลับไปได้เลย
- fallback ที่ดี: คืน object ที่ `confidence=0.0` และใส่ข้อความดิบไว้ใน `summary_th`
</details>

---

## สิ่งที่ต้องส่ง

`workshop1_extractor.py` + ผลรัน 20 ใบ + สรุปสถิติ retry
