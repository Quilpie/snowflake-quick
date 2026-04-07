# Better Together: Snowflake Semantic View & Amazon QuickSight

A Snowflake Quickstart demonstrating AI-powered BI integration between Snowflake Semantic Views and Amazon QuickSight.

## Overview

This guide shows how to:
- Set up a Snowflake warehouse, database, and schema
- Load movie review data from Amazon S3
- Create a Snowflake Semantic View with dimensions and metrics
- Use Cortex Analyst for natural language self-serve analytics
- Integrate with Amazon QuickSight via API for AI-powered BI dashboards

## Architecture

The integration leverages Snowflake's native capabilities to ingest structured data, define semantic views, and connect to Amazon QuickSight for enhanced AI-powered business intelligence.

## Prerequisites

- **Snowflake Account**: Enterprise edition with `ACCOUNTADMIN` role access
- **AWS Account**: With QuickSight subscription (US West Oregon recommended)
- Basic knowledge of SQL and Python

## Quick Start

1. Download and import the notebook into Snowsight
2. Run the Notebook to setup warehouse, database, and load data and define the semantic view `MOVIES_ANALYTICS_SV` and configure Amazon QuickSight integration and create data set

## Contents

```
├── better-together-snowflake-sv-amazon-quicksight.md  # Main guide
└── assets/                                             # Screenshots and diagrams
```

## Resources

- [Getting Started with Semantic Views](https://www.snowflake.com/en/developers/guides/snowflake-semantic-view/)
- [Getting Started with Cortex Analyst](https://www.snowflake.com/en/developers/guides/getting-started-with-cortex-analyst/)
- [Amazon QuickSight API](https://boto3.amazonaws.com/v1/documentation/api/1.12.0/reference/services/quicksight.html)

## Author

Mary Law (in partnership with AWS Ying Wang)
