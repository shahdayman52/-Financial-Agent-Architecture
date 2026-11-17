# Project Sentinel: Autonomous Financial Hedging Agent
## Architectural Proposal

**Prepared For:** Enterprise Architecture Team, NexusPay Global  
**Project:** Project Sentinel (Tier-One International Banking Client)  
**Objective:** Design an autonomous AI agent for monitoring real-time currency fluctuations and safely queuing hedging trades to mitigate risk.

---

### 1. Agent Roles
*Define the distinct responsibilities of the modular agent components within Project Sentinel.*

**Controller:**
The Controller is the always-on orchestrator and owns the continuous **Perception** phase. On a fixed polling cycle it instructs the Executor to poll `nexus_fx_market_stream` for the monitored currency pairs, intakes the returned real-time market signals (spot rates, 24-hour change, implied volatility), compares them against each client portfolio's configured volatility thresholds, and decides whether the context warrants action. When a threshold is breached, it hands the event and its context to the Planner without any human prompting, tracks the state of the run, routes tool failures to the defined fallbacks, and stops the workflow at each Human-in-the-Loop checkpoint until approval is recorded.

**Planner:**
The Planner uses the OpenAI API (ChatGPT) as its reasoning model to break a complex event, such as a sudden drop in EUR/USD, into an ordered, verifiable strategy: confirm the move with fresh market data, quantify the client's exposure, decide whether a hedge is justified, and size it. It produces a step-by-step plan that names the tool, the inputs and the expected output for each step. After each step it has the agent **Self-reflect** on the intermediate result, checking that numbers match tool outputs or the client portfolio configuration and stay within valid ranges, and revises the plan if a result is inconsistent rather than continuing on a bad assumption.

**Executor:**
The Executor carries out the Planner's steps one at a time by calling the external APIs, databases and financial tools (LangChain tool wrappers around the `tool_catalog.json` endpoints). It builds each call strictly from the tool's input schema, passes only values taken from earlier tool outputs or the client portfolio configuration supplied by the Controller, validates every response against the tool's expected output, and returns the result to the Controller. It never executes a trade; its final action is staging a trade in the pending-approval queue.

---

### 2. Architecture Comparison (Modular vs. Static)
*Explain the design choice for Project Sentinel.*

A traditional monolithic static script hard-codes one fixed path: fetch rates, apply a formula, place a trade. In a dynamic FX market that path breaks as soon as conditions change, such as a new currency pair, a different volatility regime, an updated regulatory rule or an API that is down. Each change means editing and re-testing the whole script, and a failure anywhere stops everything. A modular design built on LangChain paradigms splits the agent into a Controller, a Planner, an Executor and independent tool wrappers. Each part can be changed, tested and versioned on its own. Swapping a market data provider means replacing one tool wrapper, and changing hedging logic touches only the Planner.

The modular design also adapts better. Because the Planner reasons over the current context instead of following a fixed script, it can adjust the sequence when market conditions shift, for example by skipping a hedge when exposure is LOW or re-checking data when volatility looks abnormal. Fallbacks attach to each tool individually, so a timeout in one component triggers a defined recovery path instead of crashing the whole system. The design also scales horizontally: more portfolios or currency pairs mean more Perception cycles running through the same components, not more copies of a script.

Most importantly for a high-stakes financial setting, separating planning from execution creates natural points where the agent can **Self-reflect**. After every tool call, the Planner checks the intermediate output against the source data and the schema: is the volatility score between 0.0 and 1.0, does the hedge ratio follow from the calculated Value at Risk, is every figure in the justification traceable to a tool output or the client portfolio configuration? A static script has no such checkpoint, so a large language model inside it could **Hallucinate** a market figure or a trade size and pass it straight to execution. In the modular design, any output that fails self-reflection is rejected and the step is re-run or escalated before it can reach a trade recommendation.

---

### 3. Tool-Chaining Sequence
*Map out the step-by-step execution flow using the available enterprise tools.*

The agent chains three tools so that the output of each step becomes the input of the next, moving from market data to quantified risk to a staged trade.

1. **Tool 1: `nexus_fx_market_stream`** — During the **Perception** phase the Controller has the Executor poll this tool on a fixed cycle. When a poll breaches a threshold, the Executor makes a confirmation call with the breached `currency_pair` (e.g., `EUR/USD`) and `include_volatility_metrics: true`. It returns the spot rate, 24-hour percentage change, liquidity depth and implied volatility score (0.0–1.0). The Planner self-reflects on the payload (spot rate present, 24-hour change consistent with the spot rate, liquidity depth above zero, volatility score within 0.0–1.0) and passes the `volatility_score` forward.
2. **Tool 2: `fx_exposure_risk_calculator`** — The Executor calls this tool with the `client_portfolio_id`, the same `currency_pair` and the `volatility_score` from Tool 1. It returns `Value_at_Risk_USD`, `exposure_severity` (LOW, MEDIUM, HIGH, CRITICAL) and a recommended `hedge_ratio`. If severity is LOW, the Controller ends the run and resumes monitoring. Otherwise the Planner sizes the trade as `amount` = `hedge_ratio` × the client's exposed position in that currency pair (the baseline balance held in the client portfolio configuration that the Controller loads alongside the `client_portfolio_id` and volatility thresholds and passes to the Planner as context, so no additional tool call is needed), and sets `trade_action` to the side that offsets the direction of that exposure (e.g., `sell` for a long EUR exposure when EUR/USD is falling).
3. **Tool 3: `nexus_hedge_trade_queuer`** — After HITL Checkpoint 1 is approved, the Executor calls this tool with the `client_portfolio_id`, `trade_action`, `currency_pair`, `amount` and an `ai_justification_log` that references the market data from Tool 1, the risk assessment from Tool 2 and the exposed position from the client portfolio configuration, and records "historical precedent: not retrieved in this chain" so the risk officer can weigh precedent manually at HITL Checkpoint 1. The tool stages the trade without executing it and returns a `staging_queue_id` with status `PENDING_HUMAN_APPROVAL`.

---

### 4. Fallback Mechanisms
*Define system resilience and error handling for high-stakes banking.*

* **Fallback for Tool 1 (`nexus_fx_market_stream`):** If the call fails, times out or is rate limited, the Controller has the Executor trigger an urgent alert through `sentinel_alert_dispatcher` (`severity_level: CRITICAL`, `source_module: Perception`). That cycle produces no plan, and no cached or estimated rates are used, so the agent cannot act on stale or invented market data.
* **Fallback for Tool 2 (`fx_exposure_risk_calculator`):** If the call fails or times out, the agent returns to its previous safe state, which is passive monitoring: the partial plan is discarded, no hedge is sized, no trade is staged, and the Controller resumes its Perception cycle so the event is re-evaluated from fresh market data on the next poll.
* **Fallback for Tool 3 (`nexus_hedge_trade_queuer`):** If staging fails or times out, the Executor does not retry automatically, which avoids duplicate trades. Instead it dispatches an `ERROR` alert through `sentinel_alert_dispatcher` to the designated human portfolio manager. The alert includes the full trade payload and justification log so the manager can stage the trade manually.

---

### 5. Human-in-the-Loop (HITL) Checkpoints
*Identify critical safety boundaries for the autonomous agent.*

1. **HITL Checkpoint 1: Hedge strategy approval before staging.** After Tool 2 returns, and before the Executor calls `nexus_hedge_trade_queuer`, a NexusPay Global risk officer must review and approve the proposed hedge: the exposure severity, the Value at Risk, the hedge ratio and the derived trade amount. The workflow pauses until this approval is recorded. Human oversight is needed here because sizing the hedge is the most consequential judgment in the workflow. A mis-sized hedge can create new exposure instead of reducing it, and the risk officer can catch errors or context the model does not have, such as client-specific instructions or upcoming events.
2. **HITL Checkpoint 2: Trade release before the next hedge cycle.** After Tool 3 stages the trade with status `PENDING_HUMAN_APPROVAL`, a NexusPay Global compliance manager must authorize or reject that `staging_queue_id`. Until that decision is recorded, the Controller locks the portfolio and currency pair: the agent may not start a new hedge cycle or stage another trade for that pair, which prevents stacked or conflicting hedges built on an unconfirmed position. The agent itself has no permission to release or execute trades. Human authorization is required here because execution commits real client capital and is irreversible. NexusPay Global's control policy requires a named, accountable person, not an autonomous system, to sign off on trades made on a client's behalf.
