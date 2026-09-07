# Lab 4 · ดู JSON-RPC ที่วิ่งจริง

**10:15 – 10:30** (15 นาที)

---

## เป้าหมาย

เปลี่ยน MCP จากนามธรรมเป็นของที่จับต้องได้ ด้วยการดูข้อความจริงที่วิ่งระหว่าง client กับ server

---

## 1. ดูเวอร์ชันที่ใช้จริง

```bash
uv run python scripts/print_protocol_version.py
```

จดเลข `protocolVersion` ไว้ เอกสารทุกฉบับในคอร์สนี้อ้างอิงเลขที่ได้จากคำสั่งนี้ ไม่ได้ hardcode ไว้

---

## 2. ยิง JSON-RPC ด้วยมือ

รัน server แบบ HTTP:

```bash
uv run python apps/mcp-server/server.py --transport streamable-http --port 9000
```

เปิด terminal ใหม่ แล้วยิง `initialize` เอง:

```bash
Invoke-RestMethod -Uri http://localhost:9000/mcp -Method Post -ContentType "application/json" -Headers @{"Accept"="application/json, text/event-stream"} -Body '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-06-18","capabilities":{},"clientInfo":{"name":"manual","version":"1.0"}}}'
```

ดูรายการ tool:

```bash
Invoke-RestMethod -Uri http://localhost:9000/mcp -Method Post -ContentType "application/json" -Headers @{"Accept"="application/json, text/event-stream"} -Body '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'
```

เรียก tool จริง:

```bash
curl -s -X POST http://localhost:9000/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"list_devices","arguments":{}}}'
```

---

## 3. ใช้ MCP Inspector

เครื่องมือทางการสำหรับดูและทดสอบ MCP server:

```bash
npx @modelcontextprotocol/inspector python apps/mcp-server/server.py
```

เปิดเบราว์เซอร์ตามที่แจ้ง แล้วลอง:
- แท็บ **Tools** — เรียก `get_upstream_devices` ด้วย `["LPE-NBI-11","LPE-NBI-12","LPE-NBI-13"]`
- แท็บ **Resources** — อ่าน `clock://now` และ `schema://overview`
- แท็บ **Prompts** — ดูเทมเพลตที่มี
- ดู **raw JSON-RPC** ที่วิ่งจริงทุกครั้งที่กด

---

## 4. สิ่งที่ต้องสังเกต

- [ ] `initialize` ตอบกลับด้วย `protocolVersion` อะไร
- [ ] `tools/list` คืน `inputSchema` มาด้วย — **นี่คือสิ่งที่ทำให้โมเดลรู้ว่าต้องกรอกอะไร**
- [ ] `annotations` ของแต่ละ tool มี `readOnlyHint` ไหม
- [ ] `id` ในคำขอกับคำตอบตรงกัน
- [ ] `resources/read` คืนอะไรที่ต่างจาก `tools/call`

---

## 5. คำถามที่ต้องตอบได้

1. ถ้าส่ง request โดยไม่มี `id` จะเกิดอะไรขึ้น และเรียกว่าอะไร
2. `tools/list` กับ `resources/list` ต่างกันตรงไหนในเชิงการใช้งาน
3. ถ้า tool โยน exception client จะเห็นอะไร (ลองเรียก `get_device_config` ด้วยชื่อที่ไม่มีจริง)

---

## สิ่งที่ต้องส่ง

ผลลัพธ์ `initialize` + เลข `protocolVersion` + คำตอบ 3 ข้อข้างบน
