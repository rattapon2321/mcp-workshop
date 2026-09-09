# เฉลยวันที่ 3

> ⚠️ อ่านก่อนลอง = เสียโอกาสเรียนรู้ · ดูวิธีใช้ที่ [../README.md](../README.md)

**ไม่มีไฟล์เฉลยแยกในโฟลเดอร์นี้โดยตั้งใจ** — เพราะเฉลยของวันที่ 3 คือระบบที่ทำงานอยู่จริง

| โจทย์ | เฉลยอยู่ที่ |
|---|---|
| [Workshop 3A: MCP Server](../../instructions/day3/workshop3a-mcp-server.md) | `apps/mcp-server/` |
| [Workshop 3B: Agent API](../../instructions/day3/workshop3b-agent-api.md) | `apps/agent-api/main.py` |
| [Workshop 3C: Chainlit](../../instructions/day3/workshop3c-chainlit.md) | `apps/chainlit-ui/` |
| [โจทย์ที่ 5: Guardrail Red-team](../../instructions/day3/challenge5-guardrail-redteam.md) | `tests/test_guardrails.py` |
| [โจทย์ที่ 6: Cross-Service](../../instructions/day3/challenge6-cross-service-diagnosis.md) | `uv run python eval/run_eval.py` |

---

## วิธีทำ Workshop 3 ให้ได้ประโยชน์เต็มที่

โค้ดใน `apps/` ทำงานได้อยู่แล้วและเป็นตัวเดียวกับที่ container เดโมใช้
ถ้าเปิดอ่านก่อนเริ่ม จะเหลือแค่การลอก

**ทำแบบนี้แทน**

```bash
cp -r apps/mcp-server /tmp/mcp-server-reference
```

แล้วเขียนของตัวเองใน `apps/mcp-server/` โดยเริ่มจาก:

1. `server.py` + `config.py` + `db.py` — โครงพื้นฐาน
2. `tools/tickets.py` เพียง **1 tool** ให้ทำงานได้ก่อน
3. ทดสอบด้วย MCP Inspector ให้ผ่าน
4. แล้วค่อยเติมที่เหลือตาม pattern เดิม

เทียบกับ `/tmp/mcp-server-reference` ตอนจบ

---

## 5 จุดที่ควรอ่านเมื่อเทียบผลงาน

### 1. `security/guardrails.py` — ชั้นที่ prompt injection ชนะไม่ได้

```python
def assert_read_only(query: str, tool: str) -> None:
```

ฟังก์ชันนี้รันทั้งที่บัญชีฐานข้อมูลเป็น read-only อยู่แล้ว — เป็น defence in depth
บัญชีปกป้องข้อมูล ส่วนฟังก์ชันนี้ปกป้องกรณีที่มีคนตั้งค่าผิดในอนาคต และสร้าง audit log ที่อ่านรู้เรื่อง

**บั๊กจริงที่พบตอนเขียน test**: `\bpassword\b` ไม่ match `PG_PASSWORD` เพราะขีดล่างเป็น word character — รหัสผ่านหลุดออกไปกับผลลัพธ์ tool ได้ ดูวิธีแก้ที่ `_SECRET_NAME`

### 2. `clock.py` — "ตอนนี้" มาจากข้อมูล ไม่ใช่นาฬิกา

```python
def data_now(refresh: bool = False) -> datetime:
```

และ tool รับเฉพาะช่วงเวลาสัมพัทธ์ (`last_7d`) ไม่รับวันที่ absolute
เพราะ **ทุกวันที่ที่โมเดลคิดขึ้นเอง คือวันที่ที่มันมีโอกาสคิดผิด**

### 3. `tools/network.py` → `get_upstream_devices` — คำนวณให้ ไม่ใช่ให้โมเดลคิด

Tool นี้ไม่ได้คืนแค่ path แต่หา intersection ให้เลย พร้อมฟิลด์ `interpretation` เป็นภาษาคน

> **หลักการทั่วไป**: งานที่เขียนเป็นโค้ดได้แน่นอน อย่าปล่อยให้โมเดลทำ

### 4. `tools/reports.py` — allowlist ไม่ใช่ deny-list

```python
ALLOWED_SCRIPTS: dict[str, dict] = {...}
subprocess.run(argv, shell=False, timeout=30)
```

"ชื่อสคริปต์" ต้องเป็นชุดปิดที่นักพัฒนากำหนด ไม่ใช่ string ที่โมเดลส่งมา
และ `shell=False` เสมอ — `shell=True` คือการเปิดช่องให้ประกอบคำสั่งใหม่

### 5. `resources/schemas.py` — ทำไม schema เป็น Resource ไม่ใช่ Tool

โมเดลควร**อ่านก่อนเริ่มคิด** ไม่ใช่ต้องตัดสินใจว่าจะเรียกหรือไม่

ผลที่วัดได้: โมเดลที่อ่าน schema แล้วจะเลิกเดาชื่อ column และเลิกแต่งตารางที่ไม่มีอยู่

---

## ตรวจงานตัวเอง

```bash
make test -- tests/test_mcp_tools.py
```

```bash
make eval
```
