# Reflection Report: Beaver's Choice Paper Company Multi-Agent System

---

## 1. Agent Workflow Architecture

### Architecture Overview

The system uses an **Orchestrator Pattern** with one central orchestrator and three specialist worker agents, all implemented using the `smolagents` framework with `gpt-4o-mini` as the underlying LLM.

### Agent Roles and Responsibilities

**Orchestrator Agent** (`ToolCallingAgent` with `managed_agents`)
- Receives every customer request as a natural language prompt
- Parses the request to extract: items needed, quantities, order size, event type, and delivery deadline
- Delegates tasks to specialist agents in sequence: inventory check → quote generation → reorder if needed → delivery check → sale finalisation
- Returns a customer-friendly response with full pricing breakdown

**Inventory Agent** (`ToolCallingAgent`)
- Sole responsibility: stock management
- Tools: `check_inventory_for_items` (wraps `get_all_inventory()`), `reorder_stock` (wraps `create_transaction()` + `get_supplier_delivery_date()`), `get_inventory_snapshot` (wraps `get_all_inventory()`)
- Triggers reorders automatically when stock is zero or below minimum level

**Quoting Agent** (`ToolCallingAgent`)
- Sole responsibility: price quote generation
- Tools: `generate_quote` (reads inventory table + applies bulk discounts), `lookup_similar_quotes` (wraps `search_quote_history()`), `check_inventory_for_items` (pre-quote stock check)
- Applies bulk discount tiers: 0% (small), 5% (medium), 15% (large)

**Sales Agent** (`ToolCallingAgent`)
- Sole responsibility: transaction finalisation
- Tools: `finalize_sale` (wraps `create_transaction()`), `check_delivery_feasibility` (wraps `get_supplier_delivery_date()`), `get_current_cash_balance` (wraps `get_cash_balance()`)

### Architecture Decision Rationale

The Orchestrator Pattern was chosen over a Peer-to-Peer architecture because:
1. Customer requests are sequential in nature (check → quote → sell)
2. A single entry point simplifies error handling and response formatting
3. Each specialist agent has clearly non-overlapping responsibilities, preventing conflicts
4. The orchestrator can skip steps (e.g., skip sales finalisation if items are out of stock)

The `managed_agents` parameter in smolagents v1.24 was used to properly nest agents under the orchestrator, satisfying the rubric requirement for a true agentic solution rather than direct tool calls.

---

## 2. Evaluation Results

### Test Dataset

The system was evaluated against all 22 requests in `quote_requests_sample.csv`, covering a range of job types (office managers, hotel managers, school teachers, event managers), order sizes (small, medium, large), and event types (ceremony, parade, conference, party, reception, show).

### Key Observations from test_results.csv

**Cash Balance Changes:**
- Initial cash balance: $59.70 (after database seeding with $5,000 starting balance minus stock purchase costs)
- Cash balance decreases across multiple requests (requests 3, 4, 6, 7, 15) indicate reorder transactions being placed
- Requests where items were successfully matched and sold show positive cash flow changes
- The cash balance tracker correctly reflects the net of sales revenue minus stock purchase costs

**Fulfillment Analysis:**
- Requests involving catalogue items (Cardstock, Colored paper, Glossy paper, Kraft paper) were successfully quoted and fulfilled
- Requests for non-catalogue items (A4 white printer paper, A3 matte paper, balloons, streamers) could not be fulfilled — correctly communicated to customers
- At least 3 requests resulted in successful sales transactions with corresponding cash balance changes

**Strengths:**
1. The system correctly identified out-of-stock items and communicated alternatives to customers
2. Bulk discount logic (15% for large orders) was applied consistently across all fulfilled orders
3. Reorder logic automatically triggered `create_transaction()` with `stock_orders` type when inventory was depleted
4. Delivery feasibility checks correctly used `get_supplier_delivery_date()` to compare against customer deadlines

**Weaknesses:**
1. Fuzzy item name matching had gaps — customer requests for "A4 glossy paper" did not always match catalogue entry "Glossy paper"
2. Some requests timed out due to the managed agent pattern requiring multiple LLM calls per request
3. The `generate_quote` tool occasionally required multiple retry steps when `items_requested` parameter was not passed correctly by the orchestrator

---

## 3. Suggestions for Improvement

### Improvement 1: Enhanced Item Name Normalisation

**Problem:** The current fuzzy matching in `generate_quote` uses simple substring matching (`item_name.lower() in key.lower()`). This fails for cases like "A4 white printer paper" vs "Standard copy paper" or "A3 glossy paper" vs "Glossy paper".

**Solution:** Implement a dedicated `normalize_item_name()` function using either:
- A pre-built mapping dictionary: `{"a4 paper": "A4 paper", "glossy": "Glossy paper", "cardstock": "Cardstock"}`
- A semantic similarity approach using embeddings to find the closest catalogue match
- Add an LLM-powered item resolution tool that maps customer descriptions to catalogue entries before quoting

This would significantly increase order fulfillment rates and cash balance growth.

### Improvement 2: Add a Dedicated Financial Reporting Agent

**Problem:** Currently, financial reporting is handled ad-hoc by the sales agent via `get_cash_balance()` and `generate_financial_report()`. There is no agent that proactively monitors financial health or flags when cash balance drops below a threshold.

**Solution:** Add a **Financial Agent** with tools:
- `generate_financial_report()` — triggered after each batch of orders
- A `low_cash_alert()` tool that notifies when cash drops below a configurable threshold
- A `profitability_analysis()` tool that compares quote revenue against reorder costs

This would improve business intelligence capabilities and ensure the system remains financially sustainable, which is critical for a real paper supply company.

### Improvement 3: Partial Fulfilment Logic

**Problem:** When a customer orders 5 items and only 2 are in stock, the entire order is marked as unfulfilled. This is overly conservative and loses potential revenue.

**Solution:** Implement partial fulfilment in `finalize_sale()`:
- Fulfil available items immediately
- Trigger reorders for unavailable items
- Return a split response: "Items A and B fulfilled now, Items C and D will ship by [date]"
- Track partial orders in a new `pending_orders` database table

---

## 4. Technical Stack Summary

| Component | Technology |
|-----------|-----------|
| Agent Framework | smolagents v1.24 |
| LLM Model | gpt-4o-mini (via Vocareum/OpenAI) |
| Database | SQLite3 (munder_difflin.db) |
| Orchestration Pattern | Orchestrator + 3 Managed Worker Agents |
| Helper Functions Used | create_transaction, get_all_inventory, get_stock_level, get_supplier_delivery_date, get_cash_balance, generate_financial_report, search_quote_history |

---

*Submitted as part of the Udacity Multi-Agent Systems Nanodegree — Beaver's Choice Paper Company Project*
