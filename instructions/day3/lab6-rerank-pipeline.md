# Lab 6 · Retrieve → Rerank → Generate

**20 นาที**

---

## เป้าหมาย

เข้าใจว่าทำไม production ถึงมี rerank คั่นกลาง และวัดผลว่าช่วยจริงแค่ไหน

---

## Pipeline ของ production

```mermaid
flowchart LR
    Q["คำถาม"] --> E["EmbeddingGemma 300M<br/>แปลงเป็น vector"]
    E --> R["Retrieve<br/>ดึงมา 50 ชิ้น"]
    R --> RR["mxbai-rerank<br/>เหลือ 5 ชิ้น"]
    RR --> G[["Main Brain<br/>สร้างคำตอบ"]]
    G --> A["คำตอบ"]
    style RR fill:#e0ffe0,stroke:#0a0
```

---

## ทำไมต้องมีตัวที่สอง ทั้งที่ embedding ก็จัดอันดับให้แล้ว

| | Bi-encoder (embedding) | Cross-encoder (rerank) |
|---|---|---|
| วิธีทำงาน | เข้ารหัสคำถามกับเอกสาร**แยกกัน** แล้วเทียบ vector | **อ่านทั้งคู่พร้อมกัน** แล้วให้คะแนน |
| ความแม่น | ปานกลาง | **สูง** |
| ความเร็ว | เร็วมาก (ทำ index ล่วงหน้าได้) | ช้า (ต้องคำนวณทุกคู่) |
| ใช้กับ | ค้นจากทั้งคลัง | จัดอันดับผู้เข้ารอบ |

**Bi-encoder ไม่เคยเห็นคำถามกับเอกสารพร้อมกัน** จึงพลาดความสัมพันธ์ที่ต้องอ่านทั้งคู่ถึงจะเห็น

การใช้ทั้งคู่ต่อกันจึงได้ทั้งความเร็วและความแม่น

---

## ผลต่อ Hallucination

```mermaid
flowchart LR
    A["ส่ง 50 ชิ้นเข้าโมเดล"] --> A1["context ยาว<br/>ข้อมูลไม่เกี่ยวปนเยอะ<br/>→ โมเดลหยิบผิด"]
    B["ส่ง 5 ชิ้นที่ตรงจริง"] --> B1["context สั้น<br/>ตรงประเด็น<br/>→ ตอบแม่นขึ้น"]
    style A1 fill:#ffe0e0,stroke:#c00
    style B1 fill:#e0ffe0,stroke:#0a0
```

> **ส่งข้อมูลน้อยแต่ตรง ดีกว่าส่งเยอะ** — ข้อมูลที่ไม่เกี่ยวข้องไม่ได้เฉยๆ แต่ทำให้โมเดลเข้าใจผิดได้
> เชื่อมกลับ Module 2 เรื่อง content dilution

---

## สิ่งที่ต้องทำ

### 1. เปิด rerank service

```bash
docker compose -f docker/docker-compose.yml --profile llm up -d infinity
```

### 2. ต่อเข้ากับ `search_docs_semantic` apps/agent-api/agent/rerank.py

```python
passages = retrieve(query, k=50)          # ดึงกว้าง
top = await rerank(query, passages, top_k=5)   # เหลือน้อยแต่ตรง
```

### 3. วัดผล

รันคำถามชุดเดียวกัน 2 แบบ:

| | ไม่ rerank (top 5 จาก embedding) | มี rerank (50 → 5) |
|---|---|---|
| เอกสารที่ถูกต้องติดอันดับ 1 | | |
| เอกสารที่ถูกต้องอยู่ใน 5 อันดับแรก | | |
| token ที่ส่งเข้าโมเดลหลัก | | |
| เวลารวม | | |
| คำตอบถูกต้องไหม | | |

ใช้คำถาม **Q14** และ **Q17** จาก `data/questions/`

---

## เกณฑ์ผ่าน

- [ ] เรียก rerank service ได้
- [ ] ตารางเปรียบเทียบครบ
- [ ] อธิบายได้ว่า rerank ช่วยหรือไม่ช่วยในเคสไหน
- [ ] ระบบยังทำงานได้เมื่อ rerank service ล่ม (degrade เป็นลำดับเดิม)

---

## โบนัส

1. ลอง `RETRIEVE_TOP_K` ที่ 20 / 50 / 100 — ดึงกว้างขึ้นช่วยเสมอไหม
2. ทำ hybrid: ผสมคะแนน BM25 กับ vector ก่อน rerank
