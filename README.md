# orchestra-skills

Skill สำหรับสั่งงาน **claude-orchestrator** — control plane ที่รัน Claude Code
ข้ามหลาย account พร้อมกัน มีคิว มีบอร์ดสด และมี `PreToolUse` gate ที่ค้าง tool call
ไว้รอคนตอบ

```
skills/
  orchestra/SKILL.md    # dispatch, เลือก pool, เขียน prompt, fan-out, worktree, ดักบั๊กที่เคยเจอ
```

## ติดตั้ง

Claude Code โหลด skill จาก `~/.claude/skills/<name>/SKILL.md` — ทำ junction ชี้กลับมาที่ repo นี้
เพื่อให้ repo เป็นตัวจริงตัวเดียว (แบบเดียวกับ `nohell-skill`)

```powershell
# Windows (cmd) — ต้องไม่มี ~/.claude/skills/orchestra อยู่ก่อน
mklink /J "%USERPROFILE%\.claude\skills\orchestra" "%USERPROFILE%\Desktop\orchestra-skills\skills\orchestra"
```

```bash
# macOS / Linux
ln -s ~/Desktop/orchestra-skills/skills/orchestra ~/.claude/skills/orchestra
```

เรียกใช้ด้วย `/orchestra` หรือปล่อยให้ Claude โหลดเองเมื่อเจองานที่ต้องกระจายข้าม account

## ข้อจำกัดที่ต้องรู้ก่อนเอาไปใช้เครื่องอื่น

SKILL.md เล่มนี้ **ไม่ portable** — มัน hardcode ค่าของเครื่องที่เขียนไว้: path ของโฟลเดอร์
orchestrator บน Desktop, Node v26.7.0, ชื่อ pool (`team1`, `team2`, `account_c`…`account_f`,
`scratch`) และ worktree ชุด `C:/mm-*` ทั้งเล่มเขียนจากการวัดของจริงบนเครื่องนั้น ไม่ได้เขียนจาก README
ย้ายเครื่องเมื่อไหร่ต้องไล่แก้ path กับชื่อ pool ก่อน

ไม่มี token หรือ secret อยู่ในไฟล์ — `ORCHESTRATOR_TOKEN` ถูกอ่านจาก `.env` ของ orchestrator ตอนรัน
