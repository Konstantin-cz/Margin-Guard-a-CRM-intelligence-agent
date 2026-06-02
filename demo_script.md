# MCP Usage Explanation

**Loomi Connect MCP used:** `loomi-marketing` — a single server endpoint that exposes both Marketing and Analytics capabilities for the `golden-waffle` Bloomreach Engagement project. All access is read-only.

- **Data investigation:** We used the MCP to interrogate the raw event schema, validate data quality, and explore customer purchase behavior — surfacing critical data issues (outlier price contamination, string-typed boolean fields) that directly shaped the agent's query logic and calculation methodology.

- **Cohort calculation:** The agent uses `execute_analytics_eql` to query live customer data at runtime, combining pre-built Bloomreach segmentations and per-customer aggregates to identify which customers should be suppressed from the launch discount, which should receive an offer, and which should be targeted for reactivation. Every query is parameterized to the brief's specific product category — no pre-built filters, no hardcoded audiences.

- **Margin calculation:** The agent uses `execute_analytics_eql` to pull the suppression cohort's order-value distribution, computes the median transaction value (with outlier filtering), and multiplies by cohort size and the counterfactual discount percentage — producing a per-launch and annual margin-protected figure grounded in real customer data.

**The key point for judges:** `execute_analytics_eql` queries are constructed fresh on every brief, with the launch category extracted from the user's natural-language input at runtime. A "women's tops" brief and a "men's footwear" brief produce entirely different MCP calls — no pre-built filters, no hardcoded audiences. The MCP is not called in the background to retrieve a result; it is the agent's primary reasoning layer.
