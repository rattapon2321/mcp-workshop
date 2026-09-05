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

**ทำไมต้อง 768** — ต้องตรงกับมิติของโมเดล ถ้าใส่ผิด `INSERT` จะ error ทุกแถว

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
import httpx, psycopg

PG = "postgresql://mpls:mpls_dev_password@localhost:5432/mplsdb"
EMB = "http://localhost:11434/v1/embeddings"
MODEL = "embeddinggemma:300m"

def embed_batch(texts: list[str]) -> list[list[float]]:
    r = httpx.post(EMB, json={"model": MODEL, "input": texts}, timeout=60)
    r.raise_for_status()
    data = sorted(r.json()["data"], key=lambda d: d["index"])   # อย่าลืมเรียงลำดับ
    return [d["embedding"] for d in data]

with psycopg.connect(PG) as conn:
    cur = conn.cursor()
    cur.execute("SELECT ticket_id, title, description FROM tickets ORDER BY ticket_id")
    rows = cur.fetchall()

    BATCH = 32          # ยิงทีละ 1 แถวจะช้ามาก
    for i in range(0, len(rows), BATCH):
        chunk = rows[i:i+BATCH]
        vecs = embed_batch([f"{t}\n\n{d}" for _, t, d in chunk])
        for (tid, _, _), v in zip(chunk, vecs):
            cur.execute("UPDATE tickets SET embedding = %s WHERE ticket_id = %s",
                        (str(v), tid))
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

ค้นด้วย Python:

```python
q = embed_batch(["ลูกค้าบ่นว่าอินเทอร์เน็ตหลุดบ่อย"])[0]
cur.execute("""SELECT ticket_id, title, embedding <=> %s::vector AS d
               FROM tickets ORDER BY embedding <=> %s::vector LIMIT 5""",
            (str(q), str(q)))
for r in cur.fetchall(): print(r)
```

`<=>` คือ cosine distance — **ยิ่งน้อยยิ่งใกล้**

---

## ขั้นที่ 6 · ทำ Neo4j ด้วย

```cypher
CREATE VECTOR INDEX device_embedding IF NOT EXISTS
FOR (d:Device) ON (d.embedding)
OPTIONS { indexConfig: {
  `vector.dimensions`: 768,
  `vector.similarity_function`: 'cosine'
}};
```

แล้ว backfill `d.profile_text` เข้าไป (ดูตัวอย่างใน `docker/seeder/seed_neo4j.py`)

ค้นหา:

```cypher
CALL db.index.vector.queryNodes('device_embedding', 3, $vec)
YIELD node, score RETURN node.device_id, score
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
