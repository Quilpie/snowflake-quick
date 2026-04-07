---
name: snowflake-amazon-quick-skills
description: "Set up Snowflake Cortex Agent integration with Amazon QuickSight via MCP. Use when: user wants to connect agent to QuickSight, set up MCP server for Amazon, integrate Snowflake with AWS QuickSight, create sales intelligence dataset, create semantic view, create cortex search service, create cortex agent for QuickSight, enable OAuth for QuickSight. Triggers: quicksight, amazon quick, quick skills, MCP server quicksight, expose agent amazon, aws integration, oauth quicksight, sales intelligence, cortex agent quicksight."
---

# Snowflake and Amazon Quick Skills

End-to-end workflow to create a Sales Intelligence Cortex Agent and expose it to Amazon QuickSight via Snowflake Managed MCP.

## Prerequisites

- ACCOUNTADMIN role access (or equivalent privileges)
- Amazon QuickSight access with admin permissions
- A Snowflake warehouse available for compute

## Workflow

### Step 1: Gather Configuration

**Ask user for the following using the `ask_user_question` tool:**

1. DATABASE name (default: SALES_INTELLIGENCE)
2. WAREHOUSE name (default: SALES_INTELLIGENCE_WH)
3. AWS_REGION — **MUST ask the user to verify which AWS region their Amazon QuickSight is deployed in.** Present these options:
   - `us-east-1` (US East - N. Virginia)
   - `us-west-2` (US West - Oregon)
   - `eu-west-1` (EU - Ireland)
   - `eu-west-2` (EU - London)
   - `ap-southeast-2` (Asia Pacific - Sydney)

   The region is critical because it determines the OAuth redirect URI (`https://<AWS_REGION>.quicksight.aws.amazon.com/sn/oauthcallback`). An incorrect region will cause OAuth to fail.

**Use defaults:**
- `<SCHEMA>` = `DATA`
- `<ACCOUNT>` = read from the workspace execution context

**Store values as:**
- `<DATABASE>` - Database name
- `<SCHEMA>` - Schema name (DATA)
- `<WAREHOUSE>` - Warehouse name
- `<ACCOUNT>` - Account identifier (from context)
- `<AWS_REGION>` - QuickSight AWS region (user-verified)

**Stop**: Confirm all values with the user before proceeding.

---

### Step 2: Create Database, Schema, Warehouse, and Tables

**Execute:**

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE DATABASE <DATABASE>;
CREATE OR REPLACE SCHEMA <DATABASE>.<SCHEMA>;
CREATE OR REPLACE WAREHOUSE <WAREHOUSE>;

CREATE TABLE <DATABASE>.<SCHEMA>.SALES_CONVERSATIONS (
    conversation_id VARCHAR,
    transcript_text TEXT,
    customer_name VARCHAR,
    deal_stage VARCHAR,
    sales_rep VARCHAR,
    conversation_date TIMESTAMP,
    deal_value FLOAT,
    product_line VARCHAR
);

CREATE TABLE <DATABASE>.<SCHEMA>.SALES_METRICS (
    deal_id FLOAT PRIMARY KEY,
    customer_name VARCHAR,
    deal_value FLOAT,
    close_date DATE,
    sales_stage VARCHAR,
    win_status BOOLEAN,
    sales_rep VARCHAR,
    product_line VARCHAR
);
```

Then insert sample data into both tables. Refer to the notebook `/B2T/Snowflake_and_Amazon_Quick_Skills.ipynb` cell "Insert Sample Data" for the full INSERT statements with 10 conversation records and 15 metrics records.

---

### Step 3: Create Cortex Search Service

**Execute:**

```sql
CREATE OR REPLACE CORTEX SEARCH SERVICE <DATABASE>.<SCHEMA>.SALES_CONVERSATION_SEARCH
  ON transcript_text
  ATTRIBUTES conversation_id, customer_name, deal_stage, sales_rep, conversation_date, deal_value, product_line
  WAREHOUSE = <WAREHOUSE>
  TARGET_LAG = '1 hour'
  AS (
    SELECT
      conversation_id,
      transcript_text,
      customer_name,
      deal_stage,
      sales_rep,
      conversation_date,
      deal_value,
      product_line
    FROM <DATABASE>.<SCHEMA>.SALES_CONVERSATIONS
  );
```

> **Note:** `TARGET_LAG` is required. The `ON` and `ATTRIBUTES` clauses must come before `WAREHOUSE` and `TARGET_LAG`.

---

### Step 4: Create Semantic View

**Execute:**

```sql
CREATE OR REPLACE SEMANTIC VIEW <DATABASE>.<SCHEMA>.SALES_METRICS_MODEL
  TABLES (
    sales_metrics AS <DATABASE>.<SCHEMA>.SALES_METRICS
      PRIMARY KEY (DEAL_ID)
      COMMENT = 'Sales metrics for deal tracking and pipeline analysis'
  )
  DIMENSIONS (
    sales_metrics.customer_name_dim AS CUSTOMER_NAME
      COMMENT = 'Name of the customer organization',
    sales_metrics.close_date_dim AS CLOSE_DATE
      COMMENT = 'Expected or actual closing date of the deal',
    sales_metrics.sales_stage_dim AS SALES_STAGE
      COMMENT = 'Current stage in the sales pipeline (Discovery, Proposal, Technical Review, Negotiation, Closing, Expansion, At Risk, Closed Won, Closed Lost)',
    sales_metrics.win_status_dim AS WIN_STATUS
      COMMENT = 'Whether the deal was won (TRUE) or not (FALSE)',
    sales_metrics.sales_rep_dim AS SALES_REP
      COMMENT = 'Name of the sales representative managing the deal',
    sales_metrics.product_line_dim AS PRODUCT_LINE
      COMMENT = 'Product category (Enterprise Suite, Professional, Starter Package)'
  )
  METRICS (
    sales_metrics.total_deal_value AS SUM(DEAL_VALUE)
      COMMENT = 'Total deal value in USD',
    sales_metrics.avg_deal_value AS AVG(DEAL_VALUE)
      COMMENT = 'Average deal value in USD',
    sales_metrics.deal_count AS COUNT(DEAL_ID)
      COMMENT = 'Number of deals'
  )
  COMMENT = 'Sales metrics model for deal tracking and pipeline analysis';
```

> **Note:** Semantic views use `TABLES`, `DIMENSIONS`, and `METRICS` top-level clauses — NOT the `AS SEMANTIC MODEL OF ... WITH` shorthand.

---

### Step 5: Grant Usage Permissions

**Review grants with user before executing:**

```sql
GRANT USAGE ON DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT DATABASE ROLE SNOWFLAKE.CORTEX_AGENT_USER TO ROLE PUBLIC;
GRANT USAGE ON ALL SCHEMAS IN DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT SELECT ON <DATABASE>.<SCHEMA>.SALES_METRICS TO ROLE PUBLIC;
GRANT USAGE ON CORTEX SEARCH SERVICE <DATABASE>.<SCHEMA>.SALES_CONVERSATION_SEARCH TO ROLE PUBLIC;
GRANT SELECT ON ALL SEMANTIC VIEWS IN DATABASE <DATABASE> TO ROLE PUBLIC;
GRANT USAGE ON WAREHOUSE <WAREHOUSE> TO ROLE PUBLIC;
```

**Stop**: Get explicit approval before granting. User may prefer a custom role instead of PUBLIC.

---

### Step 6: Create Cortex Agent (QUICK_SALES_AGENT)

**Execute:**

```sql
CREATE OR REPLACE AGENT <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT
  COMMENT = 'Sales Intelligence Agent for Amazon QuickSight - combines structured sales metrics and unstructured conversation data'
  FROM SPECIFICATION $$
{
  "models": {
    "orchestration": "auto"
  },
  "orchestration": {},
  "instructions": {
    "response": "make the response concise and direct so that a strategic sales person can quickly understand the information provided. Use visual/chart if it helps to demonstrate",
    "orchestration": "use the analyst tool for sales metric and the search tool for call and meeting details",
    "sample_questions": [
      {"question": "what is the meeting with TechCorp Inc about and any action item?"},
      {"question": "List our sales deal in descending order"}
    ]
  },
  "tools": [
    {
      "tool_spec": {
        "type": "cortex_analyst_text_to_sql",
        "name": "SALES_METRICS_MODEL"
      }
    },
    {
      "tool_spec": {
        "type": "cortex_search",
        "name": "SALES_CONVERSATION_SEARCH",
        "description": "Search tool for sales conversation transcripts and meeting details"
      }
    }
  ],
  "tool_resources": {
    "SALES_METRICS_MODEL": {
      "semantic_view": "<DATABASE>.<SCHEMA>.SALES_METRICS_MODEL",
      "execution_environment": {
        "type": "warehouse",
        "warehouse": "<WAREHOUSE>",
        "query_timeout": 600
      }
    },
    "SALES_CONVERSATION_SEARCH": {
      "search_service": "<DATABASE>.<SCHEMA>.SALES_CONVERSATION_SEARCH",
      "max_results": 4,
      "id_column": "CONVERSATION_ID",
      "title_column": "CUSTOMER_NAME"
    }
  }
}
$$;

GRANT USAGE ON AGENT <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT TO ROLE PUBLIC;
```

**Verify:**
```sql
DESC AGENT <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT;
```

**Test the agent:**
```sql
SELECT TRY_PARSE_JSON(
  SNOWFLAKE.CORTEX.DATA_AGENT_RUN(
    '<DATABASE>.<SCHEMA>.QUICK_SALES_AGENT',
    $${ "messages": [{ "role": "user", "content": [{ "type": "text", "text": "List our sales deals in descending order by deal value" }] }] }$$
  )
) AS response;
```

---

### Step 7: Create MCP Server

**Execute:**

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE MCP SERVER <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT_MCP_SERVER
  FROM SPECIFICATION $$
    tools:
      - title: "QUICK_SALES_AGENT"
        name: "QUICK_SALES_AGENT"
        type: "CORTEX_AGENT_RUN"
        identifier: "<DATABASE>.<SCHEMA>.QUICK_SALES_AGENT"
        description: "Cortex Agent exposed via MCP for Amazon QuickSight integration"
  $$;

GRANT USAGE ON MCP SERVER <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT_MCP_SERVER TO ROLE PUBLIC;
```

**Verify:**
```sql
SHOW MCP SERVERS IN SCHEMA <DATABASE>.<SCHEMA>;
```

---

### Step 8: Create OAuth Integration for Amazon QuickSight

The redirect URI format is `https://<AWS_REGION>.quicksight.aws.amazon.com/sn/oauthcallback`.

**Execute:**

```sql
USE ROLE ACCOUNTADMIN;

CREATE OR REPLACE SECURITY INTEGRATION SA_QUICK_OAUTH
    TYPE = OAUTH
    ENABLED = TRUE
    OAUTH_CLIENT = CUSTOM
    OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
    OAUTH_REDIRECT_URI = 'https://<AWS_REGION>.quicksight.aws.amazon.com/sn/oauthcallback'
    OAUTH_ISSUE_REFRESH_TOKENS = TRUE
    PRE_AUTHORIZED_ROLES_LIST = ('PUBLIC')
    OAUTH_REFRESH_TOKEN_VALIDITY = 86400;

GRANT USAGE ON INTEGRATION SA_QUICK_OAUTH TO ROLE PUBLIC;
```

---

### Step 9: Retrieve OAuth Credentials

**Execute:**

```sql
DESC INTEGRATION SA_QUICK_OAUTH;
```

```sql
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('SA_QUICK_OAUTH');
```

**Present to user:**

```
Save these credentials for Amazon QuickSight configuration:

OAUTH_CLIENT_ID:     <value from query>
OAUTH_CLIENT_SECRET: <value from query>

Key URLs:
  MCP Server URL:    https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/QUICK_SALES_AGENT_MCP_SERVER
  Authorization URL: https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize
  Token URL:         https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request
  Refresh URL:       https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request
  Scopes:            session:role:PUBLIC
```

**Stop**: Ensure user has copied credentials and URLs before proceeding.

---

### Step 10: Configure Amazon QuickSight (Manual Steps)

**Present these instructions to the user:**

```
AMAZON QUICKSIGHT CONFIGURATION
================================

Ref: https://github.com/sfc-gh-mmarzillo/cortex-mcp-quick-suite

1. Open Amazon QuickSight console

2. Navigate to manage connections / add MCP data source

3. Fill in:
   Server URL:         https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/QUICK_SALES_AGENT_MCP_SERVER
   Authentication:     OAuth 2.0
   Client ID:          <OAUTH_CLIENT_ID from Step 9>
   Client secret:      <OAUTH_CLIENT_SECRET from Step 9>
   Authorization URL:  https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize
   Token URL:          https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request
   Scopes:             session:role:PUBLIC

4. Test the connection and save
```

---

## Quick Reference

### Generated Object Names

| Object | Name |
|--------|------|
| Database | `<DATABASE>` |
| Schema | `<DATABASE>.<SCHEMA>` |
| Warehouse | `<WAREHOUSE>` |
| Sales Conversations Table | `<DATABASE>.<SCHEMA>.SALES_CONVERSATIONS` |
| Sales Metrics Table | `<DATABASE>.<SCHEMA>.SALES_METRICS` |
| Cortex Search Service | `<DATABASE>.<SCHEMA>.SALES_CONVERSATION_SEARCH` |
| Semantic View | `<DATABASE>.<SCHEMA>.SALES_METRICS_MODEL` |
| Cortex Agent | `<DATABASE>.<SCHEMA>.QUICK_SALES_AGENT` |
| MCP Server | `<DATABASE>.<SCHEMA>.QUICK_SALES_AGENT_MCP_SERVER` |
| OAuth Integration | `SA_QUICK_OAUTH` |

### Key URLs (replace placeholders)

| URL | Value |
|-----|-------|
| MCP Server URL | `https://<ACCOUNT>.snowflakecomputing.com/api/v2/databases/<DATABASE>/schemas/<SCHEMA>/mcp-servers/QUICK_SALES_AGENT_MCP_SERVER` |
| Authorization URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/authorize` |
| Token URL | `https://<ACCOUNT>.snowflakecomputing.com/oauth/token-request` |
| QuickSight Redirect | `https://<AWS_REGION>.quicksight.aws.amazon.com/sn/oauthcallback` |

---

## Troubleshooting

### "Object does not exist" in QuickSight
- Verify MCP server exists: `SHOW MCP SERVERS IN SCHEMA <DATABASE>.<SCHEMA>;`
- Check grants: `SHOW GRANTS ON MCP SERVER <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT_MCP_SERVER;`

### OAuth errors
- Verify redirect URL matches your QuickSight region: `DESC INTEGRATION SA_QUICK_OAUTH;`
- Ensure integration is enabled

### Permission denied
- Check role grants: `SHOW GRANTS TO ROLE PUBLIC;`
- Verify warehouse access: `SHOW GRANTS ON WAREHOUSE <WAREHOUSE>;`

### Agent not responding
- Test agent directly first via `DATA_AGENT_RUN`
- Check search service is active: `SHOW CORTEX SEARCH SERVICES IN SCHEMA <DATABASE>.<SCHEMA>;`
- Check semantic view exists: `SHOW SEMANTIC VIEWS IN SCHEMA <DATABASE>.<SCHEMA>;`

---

## Cleanup (if needed)

```sql
DROP MCP SERVER IF EXISTS <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT_MCP_SERVER;
DROP AGENT IF EXISTS <DATABASE>.<SCHEMA>.QUICK_SALES_AGENT;
DROP SECURITY INTEGRATION IF EXISTS SA_QUICK_OAUTH;
DROP CORTEX SEARCH SERVICE IF EXISTS <DATABASE>.<SCHEMA>.SALES_CONVERSATION_SEARCH;
DROP SEMANTIC VIEW IF EXISTS <DATABASE>.<SCHEMA>.SALES_METRICS_MODEL;
DROP TABLE IF EXISTS <DATABASE>.<SCHEMA>.SALES_CONVERSATIONS;
DROP TABLE IF EXISTS <DATABASE>.<SCHEMA>.SALES_METRICS;
DROP SCHEMA IF EXISTS <DATABASE>.<SCHEMA>;
DROP DATABASE IF EXISTS <DATABASE>;
DROP WAREHOUSE IF EXISTS <WAREHOUSE>;
```

---

## Stopping Points

- After Step 1: Confirm configuration values
- After Step 5: Review permissions before granting (security checkpoint)
- After Step 6: Verify agent works with test query
- After Step 9: Ensure credentials are saved
- After Step 10: Confirm QuickSight connection works

## Output

- Sales Intelligence dataset (conversations + metrics)
- Cortex Search Service for unstructured data
- Semantic View for structured analytics
- Cortex Agent combining both tools
- MCP Server exposing the agent
- OAuth integration for Amazon QuickSight
- Working connection between QuickSight and Snowflake Agent
