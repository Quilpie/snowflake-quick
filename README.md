# snowflake-quick

This repo uses Cortex Code to set up a demo of a Snowflake environment — including warehouse and database — and creates a Cortex Agent with an MCP Server secured via OAuth to connect to Amazon QuickSight.

---

## What is Cortex Code?

[Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code) is Snowflake's AI-powered coding assistant. It is available directly in **Snowsight** (no setup required) and as a **CLI** for local terminal use. Both support guided workflows called **skills**.

---

## What This Skill Does

This skill walks you through building a **Sales Intelligence demo** that showcases how Snowflake's AI capabilities can power natural language analytics inside Amazon QuickSight.

The skill loads sample sales data — conversation transcripts and deal metrics — into Snowflake, then builds a **Cortex Agent** that can answer questions across both structured and unstructured data. The agent is exposed via a **Snowflake Managed MCP Server** and secured with an **OAuth integration**, giving Amazon QuickSight a direct, authenticated connection to query it using natural language.

At the end you will have a working end-to-end flow where a QuickSight user can ask questions like *"What are our top deals by value?"* or *"What was discussed in the TechCorp meeting?"* and get answers driven by Snowflake AI.

The skill guides you through each step interactively — confirming configuration, reviewing permissions, and verifying each object before moving on.

### How it works

```
Amazon QuickSight
      │  OAuth 2.0
      ▼
MCP Server  ──────────────────────────────────────▶  Cortex Agent
                                                         │         │
                                              Cortex Analyst   Cortex Search
                                              (deal metrics)   (call transcripts)
                                                   │                 │
                                             Semantic View     SALES_CONVERSATIONS
                                             SALES_METRICS          table
                                                 table
```

---

## Using the Quick Skill

### In Snowsight (no install required)

Cortex Code is built into Snowsight — open any Workspace and start chatting. To add this skill as a personal skill in your workspace:

1. In your Workspace, create the directory `.snowflake/cortex/skills/snowflake-amazon-quick-skills/`
2. Add a file named `SKILL.md` with the contents from [`snowflake-amazon-quick-skills/SKILL.md`](snowflake-amazon-quick-skills/SKILL.md) in this repo
3. Invoke it by typing `/` in the Cortex Code message box and selecting **snowflake-amazon-quick-skills**

> Personal skills in Snowsight are workspace-scoped and invoked with `/`.

---

### In the CLI

Install the CLI:

```bash
pip install snowflake-cli
cortex
```

**Option 1: Add as a Remote Skill (recommended)**

Add the following to `~/.snowflake/cortex/skills.json`:

```json
{
  "remote": [
    {
      "source": "https://github.com/Quilpie/snowflake-quick",
      "ref": "main",
      "skills": [{ "name": "snowflake-amazon-quick-skills" }]
    }
  ]
}
```

Cortex Code will sync the skill automatically on next start.

**Option 2: Install Manually**

```bash
mkdir -p ~/.snowflake/cortex/skills/snowflake-amazon-quick-skills
curl -o ~/.snowflake/cortex/skills/snowflake-amazon-quick-skills/SKILL.md \
  https://raw.githubusercontent.com/Quilpie/snowflake-quick/main/snowflake-amazon-quick-skills/SKILL.md
```

**Invoke the Skill**

Once installed, tag it using the `$` prefix:

```
$snowflake-amazon-quick-skills
```

Cortex Code will guide you step-by-step through the full setup.

---

## What the Skill Sets Up

| Object | Description |
|--------|-------------|
| Database + Schema | Snowflake environment for sales intelligence data |
| Warehouse | Compute resource |
| Tables | `SALES_CONVERSATIONS` and `SALES_METRICS` with sample data |
| Cortex Search Service | For unstructured conversation data |
| Semantic View | For structured sales metrics analytics |
| Cortex Agent | Combines search and analyst tools |
| MCP Server | Exposes the agent via Snowflake Managed MCP |
| OAuth Integration | Secures the connection to Amazon QuickSight |
