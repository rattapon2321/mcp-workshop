# 00 · สิ่งที่ต้องเตรียมก่อนอบรม

> ส่งให้ผู้เรียนล่วงหน้าอย่างน้อย **5 วัน** ก่อนวันอบรม
> ประสบการณ์จาก workshop ลักษณะนี้: ปัญหาที่กินเวลามากที่สุดคือ environment ไม่พร้อม ไม่ใช่เนื้อหายาก

---

## 1. พื้นฐานที่ควรมี

| หัวข้อ | ระดับที่ต้องการ |
|---|---|
| Python | เขียน class, function, `async`/`await`, virtualenv ได้ |
| Git / CLI | clone, branch, รันคำสั่งใน terminal ได้คล่อง |
| REST API / JSON | เคยเรียก API และอ่าน JSON เป็น |
| SQL | `SELECT`, `JOIN`, `GROUP BY` |
| เครือข่าย | เข้าใจ router, interface, routing protocol ระดับพื้นฐาน |

**ไม่จำเป็นต้องมี**: ประสบการณ์ LLM, ประสบการณ์ Neo4j/Cypher, ประสบการณ์ MCP

---

## 2. ติดตั้งล่วงหน้า

```bash
python3 --version   # ต้อง 3.11 ขึ้นไป
docker --version
git --version
```

| เครื่องมือ | หมายเหตุ |
|---|---|
| **Python 3.11+** | 3.12 ดีที่สุด |
| **uv** |* powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex" *|
| **Docker Desktop** | ต้องจัดสรร RAM อย่างน้อย **8 GB** ให้ Docker |
| **Git** | |
| **VS Code หรือ Cursor** | วันที่ 3 ใช้ Cursor ทดสอบต่อ MCP |
| **Claude Desktop หรือ Claude Code** | วันที่ 3 ใช้ทดสอบต่อ MCP |

### ตั้งค่า RAM ของ Docker

ระบบเปิด PostgreSQL + Neo4j + OpenSearch + Dashboards พร้อมกัน ถ้าให้ RAM น้อยเกินไป OpenSearch จะ restart วนไม่จบ

Docker Desktop → Settings → Resources → Memory → **อย่างน้อย 8 GB**

---

## 3. ทดสอบการเชื่อมต่อ LLM (สำคัญที่สุด)

ทีมงานจะแจ้ง URL และ key ของ LLM ภายในให้ **กรุณาทดสอบล่วงหน้าจากเครื่องที่จะใช้จริง**

```bash
curl -s $LLM_BASE_URL/models -H "Authorization: Bearer $LLM_API_KEY" | head -40
```

ถ้าต้องต่อ VPN ให้ทดสอบขณะต่อ VPN

> เป็นจุดที่ workshop สะดุดบ่อยที่สุด — ถ้ายิงไม่ผ่าน แจ้งทีมงานก่อนวันอบรม

---

## 4. เตรียม repo และดึง image ล่วงหน้า

```bash
git clone https://github.com/LLM-CE-KMITL/mcp-workshop.git && cd mcp-workshop
```

```bash
cp .env.example .env && make install
```

```bash
docker compose -f docker/docker-compose.yml pull
```

การ pull image ล่วงหน้าประหยัดเวลาเช้าวันแรกได้ประมาณ 15-20 นาทีต่อคน

---

## 5. อ่านล่วงหน้า (ไม่บังคับ แต่ช่วยมาก)

| เรื่อง | ทำไม | เวลา |
|---|---|---|
| Tokenization และ Embedding เบื้องต้น | เริ่ม Module 1 ได้เร็วขึ้น | 20 นาที |
| [modelcontextprotocol.io](https://modelcontextprotocol.io) หน้า Introduction | เห็นภาพ Tools / Resources / Prompts | 15 นาที |
| `data/scenarios.md` | **ห้ามอ่าน** — เป็นเฉลยของโจทย์ | — |

---

## 6. แบบสอบถามก่อนอบรม

กรุณาตอบเพื่อให้วิทยากรปรับความเร็วการสอนได้เหมาะสม

1. ประสบการณ์ Python (ปี)
2. เคยเรียก LLM API ด้วยโค้ดตัวเองไหม
3. เคยเขียน AI Agent ไหม
4. เคยได้ยิน MCP มาก่อนไหม
5. งานที่ทำอยู่เกี่ยวกับโครงข่าย IP-MPLS อย่างไร

---

## 7. เช็คลิสต์ก่อนวันอบรม

- [ ] Python 3.11+ ติดตั้งแล้ว
- [ ] Docker ทำงานได้ และตั้ง RAM ≥ 8 GB
- [ ] `make install` ผ่าน
- [ ] `docker compose pull` เสร็จแล้ว
- [ ] ยิง `curl` ไปที่ LLM endpoint ได้ผลลัพธ์
- [ ] ติดตั้ง Claude Desktop / Cursor แล้ว
- [ ] ตอบแบบสอบถามแล้ว
