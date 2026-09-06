# Module 4 · วิธีคิดของ Agent (ReAct Pattern)

**09:00 – 10:30** · เป้าหมาย: เข้าใจวงจรการคิดของ agent และการจัดการความจำ

---

## 1. Agent ต่างจากการเรียก LLM ธรรมดาอย่างไร

```mermaid
flowchart LR
    subgraph P["เรียก LLM ธรรมดา"]
        A1["คำถาม"] --> A2[["LLM"]] --> A3["คำตอบ"]
    end
    subgraph R["Agent"]
        B1["เป้าหมาย"] --> B2["คิด"]
        B2 --> B3["ลงมือ"]
        B3 --> B4["สังเกตผล"]
        B4 --> B5{"พอหรือยัง"}
        B5 -->|ยัง| B2
        B5 -->|พอ| B6["คำตอบ"]
    end
```

Agent = LLM + **วงจรที่ทำซ้ำได้** + **เครื่องมือ** + **สถานะ**

---

## 2. ReAct — Reasoning and Acting

```mermaid
sequenceDiagram
    participant A as Agent
    participant L as LLM
    participant T as Tool

    A->>L: เป้าหมาย + เครื่องมือที่มี + สิ่งที่รู้แล้ว
    L-->>A: Thought: ต้องรู้ก่อนว่าใครแจ้งอะไรบ้าง
    L-->>A: Action: search_tickets(range=last_14d)
    A->>T: เรียกจริง
    T-->>A: 5 ticket
    A->>L: Observation: [ผลลัพธ์]
    L-->>A: Thought: อยู่คนละ LPE ต้องหา upstream ร่วม
    L-->>A: Action: get_upstream_devices([...])
    Note over A,L: วนจนกว่าจะพอตอบ
    L-->>A: Final Answer
```

**หัวใจ**: โมเดลไม่ได้ตอบทันที แต่สลับระหว่าง *คิด* กับ *ทำ* โดยผลของการทำกลับเข้าไปเป็น input ของการคิดรอบถัดไป

---

## 3. ReAct แบบวนทีละขั้น vs วางแผนก่อนแล้วค่อยทำ

โปรเจกต์นี้ใช้แบบ **Plan-then-Execute** ซึ่งต่างจาก ReAct ดั้งเดิม

| | ReAct วนทีละขั้น | **Plan-then-Execute (ที่ใช้)** |
|---|---|---|
| เรียก LLM | ทุกขั้นตอน | 2 ครั้ง (วางแผน + สรุป) |
| ปรับตัวระหว่างทาง | ดีมาก | ต้อง re-plan |
| ทำขนานได้ | ไม่ได้ | **ได้ เพราะรู้ dependency ล่วงหน้า** |
| ตรวจสอบก่อนรัน | ไม่ได้ | **ได้ — validate plan ก่อนแตะฐานข้อมูล** |
| ผู้ใช้เห็นว่าจะทำอะไร | ไม่เห็นจนกว่าจะทำ | **เห็นแผนทั้งหมดก่อน** |

```mermaid
flowchart TB
    subgraph RE["ReAct วนทีละขั้น"]
        direction LR
        R1["LLM"] --> R2["tool"] --> R3["LLM"] --> R4["tool"] --> R5["LLM"]
    end
    subgraph PE["Plan-then-Execute"]
        direction LR
        P1["LLM<br/>วางแผน"] --> P2["tool 1"]
        P1 --> P3["tool 2"]
        P2 --> P4["tool 3"]
        P3 --> P4
        P4 --> P5["LLM<br/>สรุป"]
    end
```

> เหตุผลที่เลือก: บนโมเดล 27B ที่แชร์กัน 20 คน การเรียก LLM ทุกขั้นตอนแพงเกินไป
> และ **การให้ผู้ใช้เห็นแผนก่อนลงมือ** มีค่ามากในงานที่แตะระบบจริง

---

## 4. State Management

Agent ต้องเก็บอะไรระหว่างทำงาน

```mermaid
flowchart TB
    S["State ของหนึ่งคำถาม"] --> S1["เป้าหมายเดิม"]
    S --> S2["แผนที่วางไว้"]
    S --> S3["ผลของแต่ละขั้น"]
    S --> S4["ตัวนับสำหรับกัน loop"]
    S --> S5["สถิติ token / เวลา"]
```

ในโค้ด: `Plan`, `StepResult`, `LoopGuard`, `LLMStats` ที่ `apps/agent-api/`

**หลักการ**: state ที่ตรวจสอบได้ = ระบบที่ debug ได้ ถ้า state อยู่แต่ในหัวโมเดล เราไม่มีทางรู้ว่ามันคิดอะไรอยู่

---

## 5. Memory — สั้นและยาว

```mermaid
flowchart TB
    subgraph ST["Short-term (ในเซสชัน)"]
        A["บทสนทนาไม่กี่ turn ล่าสุด"]
        B["หัวข้อปัจจุบัน"]
        C["สรุปหัวข้อเก่าอย่างละบรรทัด"]
    end
    subgraph LT["Long-term (ข้ามเซสชัน)"]
        D["ข้อสรุปที่ควรจำ"]
        E["ความชอบของผู้ใช้"]
        F["เก็บเป็น vector ใน pgvector"]
    end
    ST -->|"สรุปสิ่งที่ควรจำ"| LT
    LT -->|"recall ตามความหมาย"| ST
```

| | Short-term | Long-term |
|---|---|---|
| อยู่ที่ไหน | RAM ของกระบวนการ | pgvector |
| อายุ | จบเซสชันก็หาย | ข้ามเซสชัน |
| ค้นอย่างไร | เรียงตามเวลา | semantic search |
| โค้ด | `apps/agent-api/agent/memory.py` | `apps/agent-api/agent/memory_longterm.py` |

> Long-term memory ใช้ **column เดียวกับที่สร้างเองใน Lab 1** — งานวันแรกกลายเป็นฐานของความจำวันที่สอง

---

## 6. ปัญหาที่ต้องแก้ให้ได้ — Context โตไม่หยุด

```mermaid
flowchart LR
    T1["turn 1<br/>800 tok"] --> T2["turn 5<br/>3,200 tok"]
    T2 --> T3["turn 10<br/>7,500 tok"]
    T3 --> T4["turn 20<br/>16,000 tok"]
    T4 --> X["ช้า · แพง ·<br/>ลืมตรงกลาง"]
    style X fill:#ffe0e0,stroke:#c00
```

**วิธีแก้ที่ผิด**: เพิ่ม context window
**วิธีแก้ที่ถูก**: ตรวจว่าเปลี่ยนเรื่องหรือยัง ถ้าเปลี่ยน ให้สรุปของเก่าเหลือบรรทัดเดียวแล้วทิ้ง detail

```mermaid
flowchart LR
    A["คุยเรื่อง NBI<br/>4 turn · 3,100 tok"] -->|"ผู้ใช้เปลี่ยนเรื่อง"| B["สรุป 1 บรรทัด<br/>120 tok"]
    B --> C["เริ่มเรื่อง BKK<br/>450 tok"]
    style B fill:#e0ffe0,stroke:#0a0
```

### สัญญาณที่ใช้ตรวจ (ถูกไปแพง)

| ลำดับ | สัญญาณ | ต้นทุน |
|---|---|---|
| 1 | ผู้ใช้พูดเองว่า "เปลี่ยนเรื่อง" | ฟรี |
| 2 | entity ที่พูดถึงเป็นคนละพื้นที่ | ฟรี |
| 3 | cosine similarity กับหัวข้อเดิมต่ำ | 1 embedding call |

### ข้อควรระวังที่สำคัญที่สุด

**คำถามนอกขอบเขตไม่ใช่การเปลี่ยนเรื่อง**

ผู้ใช้ที่ถามเรื่องร้านอาหารกลางการสืบสวน ไม่ได้เปลี่ยนหัวข้องาน การล้าง context ตรงนั้นคือการทำร้ายผู้ใช้

ดู turn ที่ 5 ใน `data/questions/L4-conversation.yaml`

---

## 7. ต่อไป

→ [Module 5: Function Calling & Tool Definition](module5-function-calling.md)
