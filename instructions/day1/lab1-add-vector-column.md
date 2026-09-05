# Lab 1 · สร้าง Vector Column ด้วยตัวเอง

**09:50 – 10:30** (40 นาที) · ต่อจาก Module 1

---

## เป้าหมาย

ทำ pipeline ของ semantic search ครบวงจรด้วยมือตัวเอง: เพิ่ม column → สร้าง embedding → backfill → สร้าง index → ค้นหา

ระบบตอนนี้ **มี embedding พร้อมใช้อยู่แล้ว** ขั้นแรกของ lab คือลบมันทิ้ง เพื่อให้ได้สร้างเอง

---

## ขั้นที่ 0 · ดูของเดิมก่อนลบ

เปิด pgAdmin (http://localhost:5050) รัน:

```sql
SELECT count(*) AS total, count(embedding) AS embedded FROM tickets;
```

ลอง semantic search ที่ Chainlit: *"เคยมีเคสเน็ตหลุดเป็นช่วงๆ ไหม"* — ทำงานได้

---

## ขั้นที่ 1 · ลบทิ้ง

```bash
cmd /c "docker exec -i mpls-postgres psql -U mpls -d mplsdb < scripts/lab/lab1_reset_vector.sql"
```

```bash
cmd /c "docker exec -i mpls-neo4j cypher-shell -u neo4j -p neo4j_dev_password < scripts/lab/lab1_reset_vector.cypher"
```

คำสั่งนี้ลบ vector ทั้งใน **PostgreSQL และ Neo4j**

ลองถามคำถามเดิมอีกครั้ง — ระบบจะตอบว่ายังไม่มี embedding

---

**ทำไมต้อง 1536** — ต้องตรงกับมิติของโมเดล ถ้าใส่ผิด `INSERT` จะ error ทุกแถว

```bash
$headers = @{
    "Authorization" = "Bearer sk-or-v1-***Your Key***"
    "Content-Type" = "application/json"
}
$body = '{"model":"openai/text-embedding-3-small","input":["test"]}'
(Invoke-RestMethod -Uri "https://openrouter.ai/api/v1/embeddings" -Method Post -Headers $headers -Body $body).data[0].embedding.Count
```

## ขั้นที่ 2 · เพิ่ม column

```sql
ALTER TABLE tickets ADD COLUMN embedding vector(1536);
```

---

## ขั้นที่ 3 · สร้าง embedding และ backfill

เขียน `my_embed.py` เอง โครงประมาณนี้:

```python
import os
import httpx
import psycopg
from dotenv import load_dotenv

load_dotenv()

PG = "postgresql://mpls:mpls_dev_password@localhost:5432/mplsdb"

# 1. เปลี่ยนตัวแปร EMB และ MODEL ให้เป็นของ OpenRouter (ตามที่คุณต้องการเปลี่ยน)
EMB = "https://openrouter.ai/api/v1/embeddings"
MODEL = "openai/text-embedding-3-small"
API_KEY = os.getenv("LLM_API_KEY", "")

def embed_batch(texts: list[str]) -> list[list[float]]:
    # 2. เพิ่ม headers สำหรับยืนยันตัวตนของ OpenRouter
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "HTTP-Referer": "http://localhost",
        "X-Title": "MCP Workshop"
    }
    
    # 3. ส่ง dimensions: 768 ไปด้วย เพื่อให้ขนาดตรงกับตารางเดิม
    r = httpx.post(
        EMB, 
        json={
            "model": MODEL, 
            "input": texts,
            "dimensions": 1536
        }, 
        headers=headers, 
        timeout=60
    )
    r.raise_for_status()
    data = sorted(r.json()["data"], key=lambda d: d["index"])   # อย่าลืมเรียงลำดับ
    return [d["embedding"] for d in data]

with psycopg.connect(PG) as conn:
    cur = conn.cursor()
    cur.execute("SELECT ticket_id, title, description FROM tickets ORDER BY ticket_id")
    rows = cur.fetchall()

    BATCH = 32           # ยิงทีละ 32 แถว
    for i in range(0, len(rows), BATCH):
        chunk = rows[i:i+BATCH]
        vecs = embed_batch([f"{t}\n\n{d}" for _, t, d in chunk])
        for (tid, _, _), v in zip(chunk, vecs):
            # แปลง vector เป็น string สำหรับใส่ใน postgres
            vector_str = "[" + ",".join(map(str, v)) + "]"
            cur.execute("UPDATE tickets SET embedding = %s::vector WHERE ticket_id = %s",
                        (vector_str, tid))
        conn.commit()
        print(f"{i+len(chunk)}/{len(rows)}")
```

### 3 จุดที่คนพลาดบ่อย

| พลาด | ผลที่เกิด |
|---|---|
| ไม่เรียง `data` ตาม `index` | vector ไปสลับ ticket กัน — **ค้นแล้วผิดโดยไม่มี error** |
| ยิงทีละแถว | 120 แถวใช้เวลาหลายนาที แทนที่จะเป็นไม่กี่วินาที |
| embed แค่ `title` | ค้นเจอน้อยลงมาก เพราะรายละเอียดอยู่ใน `description` |

---

## ขั้นที่ 4 · สร้าง index

```sql
CREATE INDEX idx_tickets_embedding ON tickets
    USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64);
```

**สร้าง index หลัง backfill เสมอ** — HNSW ที่สร้างบนตารางว่างแล้วค่อยเติมทีละแถวจะได้กราฟที่คุณภาพแย่กว่าและช้ากว่า

---

## ขั้นที่ 5 · ทดสอบ

```sql
SELECT count(*) AS total, count(embedding) AS embedded FROM tickets;
```

ค้นด้วย Python เขียน `cosine.py` เอง โครงประมาณนี้:

```python
import httpx
from openai import OpenAI

print("กำลังเชื่อมต่อ OpenRouter เพื่อสร้าง Embedding...")

# 1. ตั้งค่าเชื่อมต่อ OpenRouter (ใช้คีย์ที่ถูกต้อง)
client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key="sk-or-v1-***Your Key***",  # คีย์ที่ใช้งานได้จริง
    http_client=httpx.Client(verify=False)
)

def embed_batch(texts: list[str]) -> list[list[float]]:
    response = client.embeddings.create(
        model="openai/text-embedding-3-small",
        input=texts
    )
    return [item.embedding for item in response.data]

# 2. ทดสอบแปลงข้อความ
q = embed_batch(["ลูกค้าบ่นว่าอินเทอร์เน็ตหลุดบ่อย"])[0]

print("✅ แปลง Embedding สำเร็จ!")
print("ความยาวมิติเวกเตอร์:", len(q)) # ควรจะได้ 1536
print("ตัวอย่างข้อมูลเวกเตอร์ 5 ค่าแรก:", q[:5])
```

`<=>` คือ cosine distance — **ยิ่งน้อยยิ่งใกล้**

---

## ขั้นที่ 6 · ทำ Neo4j ด้วย

```cypher
CREATE VECTOR INDEX device_embedding IF NOT EXISTS
FOR (d:Device) ON (d.embedding)
OPTIONS { indexConfig: {
  `vector.dimensions`: 1536,
  `vector.similarity_function`: 'cosine'
}};
```

แล้ว backfill `d.profile_text` เข้าไป (ดูตัวอย่างใน `docker/seeder/seed_neo4j.py`)

ลบ Vector ขนาด 768

```cypher
DROP INDEX device_embedding IF EXISTS;
```

"สร้างใหม่" ให้รองรับ 1536 มิติ

```cypher
CREATE VECTOR INDEX device_embedding IF NOT EXISTS
FOR (d:Device) ON (d.embedding)
OPTIONS {
  indexConfig: {
    `vector.dimensions`: 1536,
    `vector.similarity_function`: 'cosine'
  }
};
```

รันไฟล์ embed_devices.py

```cypher
uv run python scripts/embed_devices.py
```

ค้นหา:

```cypher
MATCH (d:Device) 
WHERE d.embedding IS NOT NULL
WITH d.embedding AS mock_vec LIMIT 1
CALL db.index.vector.queryNodes('device_embedding', 3, mock_vec)
YIELD node, score 
RETURN node.device_id, score
```

---

## เกณฑ์ผ่าน

- [ ] `SELECT count(embedding) FROM tickets` = จำนวน ticket ทั้งหมด
- [ ] มี HNSW index บน `tickets.embedding`
- [ ] ค้นคำว่า *"เน็ตหลุดบ่อย"* แล้วได้ ticket ประเภท `intermittent` ติดอันดับต้น
- [ ] Neo4j มี vector index และทุก `:Device` มี `embedding`
- [ ] Chainlit ตอบคำถาม semantic ได้อีกครั้ง

---

## โบนัส

1. **เทียบ keyword กับ semantic** — ค้นคำว่า `circuit drop` ด้วย `LIKE` เทียบกับ vector ผลต่างกันแค่ไหน
2. **ลองใช้ `vector_l2_ops` แทน cosine** แล้วดูว่าอันดับเปลี่ยนไปอย่างไร
3. **วัดเวลา** — เทียบ query ก่อนและหลังสร้าง index

<details>
<summary>ติดเกิน 10 นาทีแล้วกดดู</summary>

- DDL เต็มอยู่ที่ `scripts/lab/lab1_solution_vector.sql` (`make lab1-solution`)
- โค้ด backfill อ้างอิงอยู่ที่ `scripts/embed_tickets.py` (`make embed-tickets`)
- ถ้า endpoint ต่อไม่ได้ ให้ใช้ `LLM_MODEL_FAST` และตรวจ VPN
</details>

---

## สิ่งที่ต้องส่ง

`my_embed.py` ของตัวเอง + ผลลัพธ์ query ขั้นที่ 5
