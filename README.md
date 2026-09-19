# Mildfoods Knowledge Base

Internal knowledge base for Mildfoods' ERP-equivalent system support (data integrity, schema, and remediation patterns).

Main file: `Mildfoods_KB_AI.md`

## Agents

`.claude/agents/` เก็บ subagent ของโปรเจกต์นี้

| Agent | หน้าที่ |
|---|---|
| `technical-analyst` | วิเคราะห์ทางเทคนิคสินทรัพย์ที่ผู้ใช้ระบุ (Elliott Wave + Time Cycle + Dow Theory, 3 timeframe) — วิเคราะห์อย่างเดียว ไม่คัดหุ้น · prompt v2.2 (19 Sep 2026) |

`technical-analyst` ไม่ประกาศ `tools:` ใน frontmatter จึงสืบทอดเครื่องมือทั้งหมดของ session
เพราะต้องใช้ทั้ง `mcp__MCP_mildfoods_sqlserver__*`, `mcp__remote-devices__mssql__*`,
`device_list_dir` / `device_stage_files` และ `Read` สำหรับเปิดกราฟ PNG
