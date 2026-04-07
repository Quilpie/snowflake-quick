# snowflake-quick

This repo uses Cortex Code to set up a demo of a Snowflake environment — including warehouse and database — and creates a Cortex Agent with an MCP Server secured via OAuth to connect to Amazon QuickSight.

---

## What is Cortex Code?

[Cortex Code](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code-cli) is Snowflake's AI-powered coding assistant CLI. It runs in your terminal and can execute SQL, write code, and follow guided workflows called **skills**.

Install and start it:

```bash
pip install snowflake-cli
cortex
```

---

## Using the Quick Skill

### Option 1: Add as a Remote Skill (recommended)

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

### Option 2: Install Manually

```bash
mkdir -p ~/.snowflake/cortex/skills/snowflake-amazon-quick-skills
curl -o ~/.snowflake/cortex/skills/snowflake-amazon-quick-skills/SKILL.md \
  https://raw.githubusercontent.com/Quilpie/snowflake-quick/main/snowflake-amazon-quick-skills/SKILL.md
```

### Invoke the Skill

Once installed, tag it in Cortex Code using the `$` prefix:

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
