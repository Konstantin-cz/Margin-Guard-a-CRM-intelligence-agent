# Margin Guard
### Discount Suppression Agent for Fashion Product Launches
**Team:** Nova Engagement Architects
**Hackathon:** Bloomreach Loomi Connect AI Hackathon 2026
**Track:** Track 4 — Engagement Intelligence & Lifecycle Agents

---

## What it does

Margin Guard is a conversational CRM intelligence agent that solves one specific problem: before a fashion product launch goes out, which customers should NOT receive the launch discount — and how much margin does that decision protect?

The agent takes a plain-English launch brief, reasons across live Bloomreach customer data via the Loomi Connect MCP, and produces a defensible targeting plan whose headline is the cohort to withhold from the discount. Every recommendation is backed by real customer counts and a quantified margin-protected figure in dollars.

What used to take a BI team hours of RFM analysis, category cross-referencing, and manual segmentation, Margin Guard does in under two minutes — directly against live Bloomreach data.

---

## The problem

In fashion ecommerce, every product launch faces the same default: 15% off, everyone gets it. That blanket discount goes to customers who would have paid full price anyway. You are not buying conversions. You are giving money away to your best customers and training them to wait for the next sale.

Margin Guard reframes the CRM question from "who should we target?" to "who should we NOT discount?" — a smaller, evidence-backed cohort that protects margin without sacrificing volume.

---

## Architecture

```
Business User (CRM Manager)
        │
        │ Natural-language launch brief
        ▼
Claude Desktop Chat
(Felix V.6 Project · Sonnet 4.6)
        │
        │ Agent runtime
        ▼
┌─────────────────────────────────────────────────────┐
│              Margin Guard Agent                      │
│                                                     │
│  1. Interpret brief                                 │
│  2. Resolve category taxonomy                       │
│  3. Fire cohort queries (parallel)                  │
│  4. Compute median AOV + margin impact              │
│  5. Emit Targeting Plan                             │
│  6. Iterate on user feedback                        │
└─────────────────────────────────────────────────────┘
        │
        │ Read-only MCP calls
        ▼
┌─────────────────────────────────────────────────────┐
│         Loomi Connect Marketing MCP                  │
│         loomi-mcp-alpha.bloomreach.com/mcp           │
│                                                     │
│  · execute_analytics_eql  (primary reasoning tool)  │
│  · list_customers_in_segment                        │
│  · list_aggregates / get_aggregate                  │
│  · get_event_schema                                 │
│  · whoami / get_project_overview                    │
└─────────────────────────────────────────────────────┘
        │
        │ Queries against
        ▼
┌─────────────────────────────────────────────────────┐
│         Bloomreach Engagement Workspace              │
│         Project: golden-waffle                      │
│                                                     │
│  · 123,162 customer profiles                        │
│  · 1.17M events (Jan 2024+)                         │
│  · RFM segmentation                                 │
│  · Per-customer aggregates                          │
└─────────────────────────────────────────────────────┘
        │
        ▼
Human approval → Platform execution (out of scope for agent)
```

---

## Loomi Connect MCP usage

**Server:** `loomi-marketing` (single endpoint exposing both Marketing and Analytics capabilities)
**Access:** Read-only throughout

| Capability | What Margin Guard uses it for |
|---|---|
| `execute_analytics_eql` | Runtime cohort queries parameterized to the brief's launch category. Constructs SUPPRESS, DISCOUNT, and WIN-BACK cohorts fresh on every run. Also pulls order-value histograms for median AOV calculation. |
| `list_customers_in_segment` | Retrieves exact live population sizes for RFM segments (Active Repeat Buyers, Lapsed buyers, etc.) |
| `list_aggregates` | Confirms pre-built per-customer signals are present before reasoning begins |
| `get_event_schema` | Used in Phase 0 data validation to confirm property names, types, and value formats — surfaced two data quality issues that shaped the agent's methodology |
| `whoami` + `get_project_overview` | Grounds the agent in the live workspace state on every run |

The key point: `execute_analytics_eql` queries are constructed fresh on every brief, parameterized to the launch category extracted from the user's natural-language input. A "women's tops" brief and a "men's footwear" brief produce entirely different MCP calls — no pre-built filters, no hardcoded audiences.

---

## Workspace setup

### Pre-built signals in Bloomreach (golden-waffle project)

**Segmentation: RFM segmentation**
- Active Repeat Buyers
- Active single buyers (high conviction)
- Active single buyers (low conviction)
- Lapsed buyers
- Non-Buyers

**Aggregates**
- `Is_new_items_count` — count of purchase_item events where is_new = "TRUE" per customer
- `total_purchase_item_count` — count of all purchase_item events per customer
- `total_spend_ltd` — sum of purchase.total_price per customer
- `purchase_count_ltd` — count of purchase events per customer
- `days_since_last_purchase` — recency in days
- `return_count`, `return_rate`

### Data quality findings (Phase 0 investigation via MCP)

During build, the agent's MCP investigation surfaced two findings permanently encoded in the methodology:

1. `purchase_item.is_new` is stored as uppercase strings `"TRUE"` / `"FALSE"` (not booleans). All EQL queries use string equality matching.
2. `purchase.total_price` contains approximately 10% outlier contamination (suspected decimal-shift corruption). All AOV calculations filter `total_price < 500` and use median over mean.

---

## How to run

### Requirements
- Claude Desktop (claude.ai/download)
- Loomi Connect Marketing MCP connected and authenticated
- Bloomreach Engagement project access (golden-waffle or equivalent)

### Setup

1. Connect the Loomi Connect MCP to Claude Desktop by adding to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "loomi-marketing": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://loomi-mcp-alpha.bloomreach.com/mcp"
      ]
    }
  }
}
```

2. Create a new Project in Claude Desktop named "Margin Guard"

3. Paste the contents of `prompt/margin_guard_system_prompt.md` into the Project Instructions field

4. Start a new chat in the Project and send a launch brief

### Example brief

```
We're launching a new women's peplum top — a statement casual piece for
fashion-conscious women who shop the tops and blouses category regularly.
Mid-price range, designed for customers who discover trends early and convert
quickly without needing a push. We have an allowance to use up to 15% discount
vouchers to support the launch.
```

### Example output

The agent produces a structured Targeting Plan:

| Cohort | Customers | Recommendation |
|---|---:|---|
| SUPPRESS — Full-price loyalists | 661 | Full price or early access. No discount. |
| SUPPRESS — High-value browsers | 14 | First-access teaser. No discount. |
| DISCOUNT — Persuadables | 413 | Time-boxed 15% offer. |
| WIN-BACK — Lapsing category buyers | 8,863 | Exclude from launch. Later reactivation. |

**Margin impact at 15% counterfactual:** $2,936 per launch / $88,088 annually (~30 launches/year)

---

## Responsible AI

**Read-only by design.** Margin Guard has no ability to write audiences, trigger campaigns, or contact customers. Every plan is prepared for human review. The agent never executes; a person does.

**Transparent reasoning.** Every cohort recommendation carries its full evidence chain — which segment, which signal, which threshold, and the resulting customer count. No black-box scores.

**Data quality disclosure.** The agent surfaces data anomalies unprompted and adjusts its methodology rather than silently proceeding with suspect inputs. Every plan includes a Data Quality Disclosures section.

**Conservative suppression.** Customers are withheld from a discount only when there is positive evidence they convert at full price. Absence of signal never triggers suppression.

**Data stays in Bloomreach.** No personal data is extracted, stored externally, or passed to third-party systems as part of the agent's operation.

---

## Repository structure

```
margin-guard/
├── README.md                          — this file
├── prompt/
│   └── margin_guard_system_prompt.md  — the agent system prompt (paste into Claude Desktop Project)
├── docs/
│   ├── project_summary.md             — 4-sentence hackathon submission summary
│   ├── mcp_usage_explanation.md       — detailed MCP usage for judges
│   ├── responsible_ai_note.md         — responsible design considerations
│   └── demo_script.md                 — full demo video script with timestamps
└── architecture/
    └── margin_guard_architecture.svg  — architecture diagram (PNG export in submission)
```

---

## Team

**Nova Engagement Architects**
Bloomreach Loomi Connect AI Hackathon 2026
Track 4 — Engagement Intelligence & Lifecycle Agents
