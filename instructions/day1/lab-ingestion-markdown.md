# Lab เสริม · Ingestion Pipeline — เอกสาร → Markdown → Vector

**20 นาที** · ทำต่อจาก Lab 1 หรือเป็นการบ้าน

> ตรงกับหัวข้อติดตามผลครั้งที่ 1: *"การแปลงไฟล์และจัดเก็บข้อมูลโครงข่ายให้อยู่ในรูปแบบ Markdown และสร้าง Vector Embeddings"*

---

## เป้าหมาย

ทำ pipeline ที่เอกสารจริงขององค์กรจะเดินผ่านตอนขึ้น production

```mermaid
flowchart LR
    A["เอกสารต้นทาง<br/>Word / PDF / config"] --> B["แปลงเป็น Markdown"]
    B --> C["Chunk ตามหัวข้อ"]
    C --> D["นับ token<br/>ตรวจขนาด"]
    D --> E["Embed"]
    E --> F[("OpenSearch<br/>network-docs")]
```

---

## ทำไมต้อง Markdown

| รูปแบบ | ปัญหา |
|---|---|
| PDF | โครงสร้างหาย ตารางเพี้ยน ลำดับข้อความสลับ |
| Word | มี metadata ปนเยอะ แปลงยาก |
| **Markdown** | **มีหัวข้อชัดเจน → chunk ตามความหมายได้** · diff ได้ · อ่านออกทั้งคนและเครื่อง |

หัวข้อ (`##`) คือขอบเขตความหมายตามธรรมชาติ ทำให้ chunk แล้วขั้นตอนกับคำอธิบายไม่ถูกตัดขาดจากกัน

---

## 1. ดู pipeline ที่มีอยู่

`docker/seeder/seed_opensearch.py` → `_chunk_markdown()`

```python
def _chunk_markdown(text: str, max_chars: int = 900) -> list[str]:
    # แบ่งตามหัวข้อ ## ก่อน แล้วค่อยแบ่งตามความยาว
```

**หลักการ**: chunk ตามหัวข้อสำคัญกว่าเทคนิค overlap ที่ซับซ้อน
ถ้าขั้นตอนการแก้ปัญหาถูกตัดครึ่ง ต่อให้ overlap ดีแค่ไหนก็ค้นเจอไม่ครบ

---

## 2. สิ่งที่ต้องทำ

### 2.1 เพิ่มเอกสารของตัวเอง

```bash
cat > data/mock_fs/runbooks/my-runbook.md <<'EOF'
# ขั้นตอนตรวจสอบ optical power

## อาการที่ควรสงสัย
interface flap ซ้ำๆ โดยที่ config ไม่ได้เปลี่ยน

## ขั้นตอน
1. ตรวจ optical power ทั้ง TX และ RX
2. เทียบกับค่าที่ SFP รองรับ
3. ถ้าต่ำกว่าเกณฑ์ ให้เปลี่ยน SFP ก่อนเปลี่ยนสาย
EOF
```

### 2.2 เขียนสคริปต์ ingest

```python
# 1. อ่านไฟล์ .md ทั้งหมดใน data/mock_fs/
# 2. chunk ด้วย _chunk_markdown
# 3. นับ token ของแต่ละ chunk ด้วย agent/tokenizer.py
# 4. เตือนถ้า chunk ไหนเกิน 512 token
# 5. embed แล้ว index เข้า network-docs
```

### 2.3 ตรวจขนาด chunk ← ขั้นที่คนมักข้าม

```python
from agent import tokenizer
for chunk in chunks:
    n = tokenizer.count(chunk)
    if n > 512:
        print(f"WARNING: chunk ยาว {n} tokens")
```

**เชื่อมกับโจทย์ที่ 1**: chunk ที่ยาว 900 ตัวอักษรเท่ากัน จะได้ token ต่างกันมากระหว่างไทยกับอังกฤษ
การ chunk ด้วยจำนวนตัวอักษรจึงให้ผลไม่สม่ำเสมอ — **ควร chunk ด้วยจำนวน token**

### 2.4 ทดสอบว่าค้นเจอ

ถาม Chainlit: *"ถ้า interface flap ควรตรวจอะไรก่อน"*
ต้องเจอ runbook ที่เพิ่งเพิ่มเข้าไป

---

### เฉลย

```python
from __future__ import annotations

import os
import hashlib
from pathlib import Path
import sys
from opensearchpy import OpenSearch

# เพิ่ม Path ไปยัง agent-api เพื่อเรียกใช้ tokenizer และ LLM
sys.path.insert(0, str(Path(__file__).resolve().parents[2] / "apps" / "agent-api"))
from agent import tokenizer

OPENSEARCH_URL = os.getenv("OPENSEARCH_URL", "http://admin:admin_dev_password@localhost:9200")
INDEX_NAME = "network-docs"
MAX_TOKENS_PER_CHUNK = 512

def _chunk_markdown_by_tokens(text: str, max_tokens: int = 400) -> list[str]:
    """
    แบ่ง chunk ตามหัวข้อ (##) ก่อน แล้วถ้าหัวข้อใดยาวเกิน max_tokens 
    ให้ซอยย่อยด้วยการนับ Token จริง เพื่อความสม่ำเสมอของขนาด Chunk
    """
    sections = text.split("\n## ")
    chunks = []
    
    for i, section in enumerate(sections):
        if i > 0:
            section = "## " + section
            
        # ถ้า section นี้เล็กกว่ากำหนด เก็บเป็น 1 chunk ได้เลย
        if tokenizer.count(section) <= max_tokens:
            chunks.append(section.strip())
        else:
            # ถ้าใหญ่เกินไป ให้แบ่งย่อยโดยเช็กจำนวน token ทีละบรรทัดหรือย่อหน้า
            lines = section.split("\n")
            current_chunk = []
            current_count = 0
            
            for line in lines:
                line_tokens = tokenizer.count(line + "\n")
                if current_count + line_tokens > max_tokens and current_chunk:
                    chunks.append("\n".join(current_chunk).strip())
                    current_chunk = [line]
                    current_count = line_tokens
                else:
                    current_chunk.append(line)
                    current_count += line_tokens
                    
            if current_chunk:
                chunks.append("\n".join(current_chunk).strip())
                
    return [c for c in chunks if c]

def main():
    print(f"\n=== เริ่มกระบวนการ Ingest เอกสารเข้า {INDEX_NAME} ===")
    
    # 1. เชื่อมต่อ OpenSearch
    client = OpenSearch(
        hosts=[OPENSEARCH_URL],
        http_auth=("admin", "admin_dev_password"),
        verify_certs=False,
        ssl_show_warn=False
    )
    
    docs_path = Path("data/mock_fs/runbooks")
    if not docs_path.exists():
        print(f"❌ ไม่พบโฟลเดอร์ {docs_path}")
        return

    warning_count = 0
    total_chunks = 0

    # 2. อ่านไฟล์ .md ทั้งหมดใน data/mock_fs/runbooks/
    for file_path in docs_path.glob("*.md"):
        print(f"\n📄 กำลังประมวลผลไฟล์: {file_path.name}")
        content = file_path.read_text(encoding="utf-8")
        
        # 3. Chunk ข้อมูล (ใช้ Token เป็นเกณฑ์เพื่อแก้ปัญหาภาษาไทย)
        chunks = _chunk_markdown_by_tokens(content, max_tokens=MAX_TOKENS_PER_CHUNK)
        
        for idx, chunk in enumerate(chunks):
            total_chunks += 1
            
            # 4. นับ token ของแต่ละ chunk ด้วย agent/tokenizer.py
            n_tokens = tokenizer.count(chunk)
            
            # 5. เตือนถ้า chunk ไหนเกิน 512 token
            if n_tokens > MAX_TOKENS_PER_CHUNK:
                print(f"  ⚠️ WARNING: Chunk #{idx} ยาว {n_tokens} tokens (เกินเกณฑ์ {MAX_TOKENS_PER_CHUNK})")
                warning_count += 1
            else:
                print(f"  ✅ Chunk #{idx}: {n_tokens} tokens")
                
            # สร้าง Deterministic ID เพื่อป้องกันการสร้างเอกสารซ้ำ (Bonus: ตรวจของซ้ำ)
            doc_id = hashlib.sha256(f"{file_path.name}_{idx}_{chunk}".encode("utf-8")).hexdigest()
            
            # เตรียมข้อมูลสำหรับ Index เข้า OpenSearch
            doc_body = {
                "file_name": file_path.name,
                "chunk_index": idx,
                "content": chunk,
                "token_count": n_tokens
            }
            
            # ทำการ Index (Upsert ป้องกันซ้ำ)
            client.index(
                index=INDEX_NAME,
                id=doc_id,
                body=doc_body,
                refresh=True
            )

    print(f"\n========================================")
    print(f"🎉 Ingest สำเร็จทั้งหมด {total_chunks} Chunks")
    print(f"⚠️ พบบันทึกเตือน Chunk ยาวเกินเกณฑ์: {warning_count} ชิ้น")
    print(f"========================================")

if __name__ == "__main__":
    main()
```

```python
uv run scripts/ingest_docs.py
```

---
## เกณฑ์ผ่าน

- [ ] เอกสารใหม่ถูก index เข้า `network-docs`
- [ ] แต่ละ chunk มี `token_count` ที่นับด้วย tokenizer ของ Gemma ไม่ใช่ตัวอักษร
- [ ] ไม่มี chunk ไหนที่ตัดขั้นตอนขาดครึ่ง
- [ ] ค้นเจอผ่าน `search_docs_semantic`
- [ ] มีรายงานว่า chunk ยาวเกินเกณฑ์กี่ชิ้น

---

## โบนัส

1. **chunk ด้วย token แทนตัวอักษร** แล้วเทียบว่าขนาดสม่ำเสมอขึ้นแค่ไหน
2. **รองรับ PDF** — ใช้ `pypdf` แปลงเป็น Markdown แล้วเทียบว่าโครงสร้างหายไปแค่ไหน
3. **ตรวจของซ้ำ** — ถ้า ingest ไฟล์เดิมสองครั้ง ต้องไม่เกิด chunk ซ้ำ (ใช้ `_id` แบบ deterministic)
4. **incremental update** — เปลี่ยนเอกสารแล้ว ingest ใหม่เฉพาะส่วนที่เปลี่ยน

---

## สำหรับ Production

| ประเด็น | ในห้อง | ของจริง |
|---|---|---|
| ปริมาณ | 5 runbook | เอกสารโครงข่ายทั้งองค์กร |
| แหล่ง | ไฟล์ในเครื่อง | SharePoint / Confluence / NEX |
| ความถี่ | ครั้งเดียว | ต้องมี incremental |
| เวอร์ชัน | ไม่มี | ต้องรู้ว่า chunk มาจากเอกสารเวอร์ชันไหน |
