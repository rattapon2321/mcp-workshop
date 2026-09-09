# Workshop 3D · ต่อกับ Claude Desktop / Claude Code / Cursor

**14:45 – 15:00** (15 นาที)

---

## โจทย์

เอา MCP Server ที่สร้างเอง ไปต่อกับ AI Client ระดับโลก แล้วสั่งงานเป็นภาษาธรรมชาติให้ไปดึงข้อมูลจริงจากฐานข้อมูลในเครื่อง

> **นี่คือช่วงที่คนมักรู้สึกว่า "มันใช้ได้จริง"** — เพราะเป็นครั้งแรกที่เห็นเครื่องมือที่ตัวเองใช้ทุกวัน เรียกโค้ดที่ตัวเองเพิ่งเขียน

---

## 1. Claude Desktop

แก้ไฟล์ config:

| ระบบ | ที่อยู่ |
|---|---|
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |

```json
{
  "mcpServers": {
    "nt-network": {
      "command": "uv",
      "args": [
        "--directory", "/absolute/path/to/ai-mpls-workshop",
        "run", "python", "apps/mcp-server/server.py",
        "--transport", "stdio"
      ],
      "env": {
        "PG_DSN": "postgresql://mcp_reader:mcp_reader_password@localhost:5432/mplsdb",
        "NEO4J_URI": "bolt://localhost:7687",
        "NEO4J_USER": "neo4j",
        "NEO4J_PASSWORD": "neo4j_dev_password",
        "OPENSEARCH_URL": "http://localhost:9200"
      }
    }
  }
}
```
หากยังไม่มีไฟล์ให้รัน `mkdir $env:APPDATA\Claude` ตามด้วย `notepad $env:APPDATA\Claude\claude_desktop_config.json` เพื่อสร้าง

**ต้อง restart Claude Desktop** หลังแก้ config

### ข้อควรระวัง

| ปัญหา | สาเหตุ |
|---|---|
| server ไม่ขึ้นในรายการ | ต้องใช้ **absolute path** เท่านั้น |
| server ขึ้นแต่ error ทันที | มี `print()` ลง stdout — stdout เป็นของโปรโตคอล ต้อง log ลง stderr |
| ต่อฐานข้อมูลไม่ได้ | Docker ยังไม่ได้เปิด หรือ env ใน config ไม่ครบ |

---

## 2. Claude Code

```bash
claude mcp add nt-network -- uv --directory "$(pwd)" run python apps/mcp-server/server.py --transport stdio
```

```bash
claude mcp list
```

---

## 3. Cursor IDE

`.cursor/mcp.json` ในโปรเจกต์:

```json
{
  "mcpServers": {
    "nt-network": {
      "command": "uv",
      "args": ["--directory", "${workspaceFolder}", "run", "python",
               "apps/mcp-server/server.py", "--transport", "stdio"]
    }
  }
}
```
เข้าที่ตั้งค่าแล้วพิมพ์ค้นหา `Model Context Protocol` ตรงหัวข้อ Chat > Mcp > Discovery: Enabled คลิกติ๊กถูก ให้อนุญาต (Enable)

✅ Cursor workspace configuration ('.cursor/mcp.json') (cursor-workspace)
---

## 4. ทดสอบด้วยภาษาธรรมชาติ

ลองพิมพ์คำสั่งเหล่านี้ใน client ที่ต่อแล้ว:

| ระดับ | คำสั่ง |
|---|---|
| ง่าย | *"ในระบบโครงข่ายมีอุปกรณ์อะไรบ้าง"* |
| กลาง | *"ticket ที่ยังไม่ปิดของนนทบุรีมีอะไรบ้าง"* |
| **ยาก** | *"ทำไมช่วงสองสัปดาห์นี้ถึงมีลูกค้าแจ้งเน็ตหลุดซ้ำๆ หลายราย"* |
| ทดสอบ Resource | *"ช่วยอ่าน schema ของฐานข้อมูลแล้วอธิบายว่าแต่ละตารางเก็บอะไร"* |
| ทดสอบ Prompt | เลือก prompt `diagnose_repeated_complaints` จากเมนู |
| ทดสอบ guardrail | *"ลบ ticket ที่ปิดแล้วทั้งหมด"* → ต้องถูกปฏิเสธ |

---

## 5. สิ่งที่ต้องสังเกต

```mermaid
flowchart LR
    A["คำถามเดียวกัน"] --> B["Chainlit ของเรา"]
    A --> C["Claude Desktop"]
    B --> D{"ผลต่างกันไหม"}
    C --> D
```

- Claude Desktop ใช้โมเดลคนละตัวกับ Gemma 27B → **แผนที่วางอาจต่างกัน**
- Claude Desktop มี native tool calling → เรียก tool แบบวนทีละขั้น ต่างจาก plan-then-execute ของเรา
- `readOnlyHint` มีผลกับการขออนุญาต — ลองเรียก `generate_report` (ที่ `readOnlyHint: false`) แล้วดูว่า client ถามก่อนไหม
- **Tool ชุดเดียวกัน โค้ดชุดเดียวกัน ทำงานได้กับ client ที่ต่างกันสิ้นเชิง** — นี่คือคุณค่าของ MCP

---

## เกณฑ์ผ่าน

- [ ] `nt-network` ปรากฏในรายการ MCP server ของ client อย่างน้อย 1 ตัว
- [ ] สั่งงานภาษาธรรมชาติแล้ว AI ไปดึงข้อมูลจริงจากฐานข้อมูลได้
- [ ] คำถามระดับยากได้คำตอบที่มี `APE-NBI-03`
- [ ] อ่าน Resource ได้
- [ ] เรียกใช้ Prompt template ได้
- [ ] คำสั่งลบข้อมูลถูกปฏิเสธ
- [ ] เทียบผลกับ Chainlit ของตัวเองแล้วอธิบายความต่างได้

---

## ต่อไป

→ [โจทย์ที่ 6: Cross-Service Diagnosis](challenge6-cross-service-diagnosis.md)
