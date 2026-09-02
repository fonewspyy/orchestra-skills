# orchestra-skills

Skill สำหรับสั่งงาน **claude-orchestrator** — control plane ที่รัน Claude Code ข้ามหลาย account
พร้อมกัน มีคิว มีบอร์ดสด และมี `PreToolUse` gate ที่ค้าง tool call ไว้รอคนตอบ

เนื้อหาทั้งหมดมาจากการวัดของจริงบน server ที่รันอยู่ ไม่ได้เขียนจาก README ของ orchestrator

```
skills/orchestra/
  SKILL.md            # ตัว skill — ความรู้ที่ใช้ได้ทุกเครื่อง
  MACHINE.example.md  # เทมเพลตค่าเฉพาะเครื่อง
  MACHINE.local.md    # ← คุณสร้างเอง, gitignored, ไม่ข้ามเครื่อง
```

## ทำไมถึงแยกเป็นสองไฟล์

ตอนแรก skill นี้ hardcode ค่าของเครื่องที่เขียนมันไว้ทั้งเล่ม — path orchestrator, เวอร์ชัน Node,
ชื่อ pool, ชื่อ worktree, port ที่ต่อ production ลงเครื่องอื่นแล้วพังเงียบๆ เพราะทุกค่าดู "น่าเชื่อ"
แต่ผิดหมด

ตอนนี้ `SKILL.md` ไม่มีค่าเฉพาะเครื่องเหลือเลย มีแต่ **§0 บอกวิธีหาค่าเหล่านั้นเอง** แล้วอ่านผลจาก
`MACHINE.local.md` ที่แต่ละเครื่องเขียนของตัวเอง

⚠ **อย่าก๊อป `MACHINE.local.md` ข้ามเครื่อง** — ชื่อ pool, layout worktree, และ port production
คือค่าที่ "ผิดแต่ดูเหมือนถูก" ที่สุดเวลาย้ายเครื่อง เป็นคลาสของ error ที่ทำให้ dispatch ยิงผิด repo
หรือ prompt ลืมบอกว่ามี database จริงต่ออยู่

## ติดตั้ง

### 1. link เข้า `~/.claude/skills/`

```powershell
# Windows (cmd) — ต้องไม่มี ~/.claude/skills/orchestra อยู่ก่อน
mklink /J "%USERPROFILE%\.claude\skills\orchestra" "<repo>\skills\orchestra"
```

```bash
# macOS / Linux
ln -s <repo>/skills/orchestra ~/.claude/skills/orchestra
```

### 2. เขียนค่าของเครื่องนี้

```bash
cp skills/orchestra/MACHINE.example.md skills/orchestra/MACHINE.local.md
```

แล้วกรอกโดย **วัดเอง** ไม่ใช่ก๊อปจากเครื่องอื่น:

| ช่อง | หาจาก |
|---|---|
| `$ORCH` | โฟลเดอร์ที่มี `.env` ซึ่งนิยาม `ORCHESTRATOR_TOKEN` |
| `$NODE` | เวอร์ชันที่ ABI ตรงกับ `better-sqlite3` — เลขในข้อความ error คือเลขที่ต้องการ |
| pools | `GET /pools` เท่านั้น อย่าเดา |
| verify command | script ของ repo เป้าหมายเอง |
| production hazard | port/tunnel/connection string ที่ห้ามแตะ |

ถ้าเครื่องนั้นไม่มี orchestrator ก็จบแค่นั้น — skill นี้เป็น client ไม่ได้ติดตั้ง server ให้

### 3. ใช้งาน

พิมพ์ `/orchestra` หรือปล่อยให้ Claude โหลดเองเมื่อเจองานที่ต้องกระจายข้าม account

## ความปลอดภัย

ไม่มี token หรือ secret ใน repo นี้ — `ORCHESTRATOR_TOKEN` ถูกอ่านจาก `.env` ของ orchestrator
ตอนรัน และ `MACHINE.local.md` ก็ไม่ควรมี token เช่นกัน (`.gitignore` กัน `*.local.md` ไว้แล้ว)
