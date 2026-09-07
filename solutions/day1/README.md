# เฉลยวันที่ 1

> ⚠️ อ่านก่อนลอง = เสียโอกาสเรียนรู้ · ดูวิธีใช้ที่ [../README.md](../README.md)

| ไฟล์ | เฉลยของ |
|---|---|
| `lab1_embed.py` | [Lab 1: สร้าง vector column](../../instructions/day1/lab1-add-vector-column.md) |
| `workshop1_extractor.py` | [Workshop 1: JSON + Auto-retry](../../instructions/day1/workshop1-json-autoretry.md) |

โจทย์ที่ 1 และ 2 อยู่ที่ [../challenges/](../challenges/)

---

## รัน

```bash
uv run solutions/day1/lab1_embed.py
```

```bash
uv run solutions/day1/workshop1_extractor.py
```

---

## 3 จุดที่คนพลาดมากที่สุดใน Lab 1

ทั้งสามจุดถูกกำกับไว้ในโค้ดด้วยคำว่า `PITFALL` พร้อมเหตุผลตรงจุดที่เกิด

| # | พลาดอะไร | อาการ |
|---|---|---|
| 1 | ไม่เรียง response ตาม `index` | **ไม่มี error ใดๆ** แต่ vector ไปผูกกับ ticket ผิดใบ ค้นแล้วได้ผลมั่ว |
| 2 | embed แค่ `title` | ค้นเจอน้อยลงมาก เพราะอาการจริงอยู่ใน `description` |
| 3 | ยิง API ทีละแถว | 120 แถวใช้เวลาหลายนาทีแทนที่จะเป็น ~20 วินาที |

จุดที่ 1 อันตรายที่สุดเพราะระบบยังทำงานได้ปกติทุกอย่าง มีแต่คำตอบที่ผิด

---

## สิ่งที่ควรสังเกตใน Workshop 1

### `_repair_prompt()` คือหัวใจ

การ retry ซ้ำด้วย prompt เดิมมักได้ผลผิดแบบเดิม แต่การส่ง **ข้อความ error กลับไปให้โมเดลเห็น** ทำให้ครั้งที่สองกลายเป็นการแก้ไข ไม่ใช่การสุ่มใหม่

```python
{"role": "assistant", "content": raw},
{"role": "user", "content": f"ไม่ผ่านการตรวจสอบ\nข้อผิดพลาด: {error}\nส่ง JSON ที่แก้แล้วกลับมา"}
```

### Delimiter แยก "คำสั่ง" ออกจาก "ข้อมูล"

```python
{"role": "user", "content": f"<<<CONVERSATION\n{text}\n>>>CONVERSATION"}
```

เป็นการป้องกัน prompt injection ที่ได้ผลที่สุด และถูกกว่าการพยายามกรอง keyword เพราะให้กรอบแก่โมเดลว่าสิ่งที่กำลังอ่านคือข้อมูล ไม่ใช่คำสั่ง

### `model_validator` ตรวจข้ามฟิลด์

`affected_device = "LPE-NBI-11"` คู่กับ `affected_site = "BKK"` เป็นสิ่งที่ **พิสูจน์ได้ว่าผิด** ไม่ใช่แค่ไม่น่าจะใช่ เพราะรหัสอุปกรณ์มีพื้นที่อยู่ในตัวมันเอง จึงคุ้มที่จะ reject แล้วให้แก้

### Fallback ต้องไม่ throw

pipeline ที่ตายเพราะแถวเดียวเสีย แย่กว่า pipeline ที่ติดธงแล้วทำต่อ
