# Mildfoods Knowledge Base

Internal knowledge base for Mildfoods' ERP-equivalent system support (data integrity, schema, and remediation patterns).

Main file: `Mildfoods_KB_AI.md`

## Support agent

`.claude/agents/mildfoods-support.md` defines a Claude Code subagent that answers
questions about the Mildfoods system from this knowledge base and confirms them
with live read-only SQL through the `MCP_mildfoods_sqlserver` connector.

Use it from Claude Code:

```
> ask the mildfoods-support agent why production PRD... fails to approve
```

or let Claude route to it automatically when a question is about STOCKTRX,
produceitem, BOM, NeedToBuy, cost/CycleTime/Loss SPs, the P&L pipeline, the
Balance Sheet, AP/AR, lot numbering, or negative-balance remediation.

It runs in its own context window, so it can navigate the ~2,200-line KB
(section index → targeted line ranges) without filling up the main session. It
carries each fact's `[VERIFIED]` / `[PARTIAL]` / `[OPEN QUESTION]` tag into its
answer, and refuses to assert anything listed as a known gap in §15.

Requires the `MCP_mildfoods_sqlserver` MCP server to be configured for live
queries; without it the agent still answers from the KB alone.
