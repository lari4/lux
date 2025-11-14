# Lux Agent Pipelines and Workflows

This document provides comprehensive documentation of all agent workflows, pipelines, and data flows in the Lux multi-agent system. Each pipeline is documented with ASCII diagrams, prompt sequences, data transformations, and integration points.

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Trading Workflows](#trading-workflows)
3. [Chat and Conversation Workflows](#chat-and-conversation-workflows)
4. [Company and Multi-Agent Workflows](#company-and-multi-agent-workflows)
5. [Agent Collaboration Patterns](#agent-collaboration-patterns)
6. [Signal-Based Workflows](#signal-based-workflows)
7. [Memory Integration Workflows](#memory-integration-workflows)
8. [Tool Execution Pipelines](#tool-execution-pipelines)

---

## Architecture Overview

### Three-Layer Tool System

Lux uses a hierarchical tool system for building agent workflows:

```
┌─────────────────────────────────────────────────┐
│                   BEAMS                         │
│  (Complex workflows with orchestration)         │
│  - Sequential, parallel, conditional execution  │
│  - Parameter passing between steps              │
│  - Error handling with retries and fallbacks    │
└───────────────┬─────────────────────────────────┘
                │ Orchestrates
                ▼
┌─────────────────────────────────────────────────┐
│                  PRISMS                         │
│     (Single-purpose transformations)            │
│  - Modular, composable units                    │
│  - Input/output schemas                         │
│  - Can be chained in beams                      │
└───────────────┬─────────────────────────────────┘
                │ May use
                ▼
┌─────────────────────────────────────────────────┐
│                  LENSES                         │
│       (External data loading)                   │
│  - API calls, database queries                  │
│  - Authentication support                       │
│  - Return data to calling agent                 │
└─────────────────────────────────────────────────┘
```

### LLM Integration Layer

```
┌──────────────┐
│    Agent     │
│  (has goal,  │
│ capabilities)│
└──────┬───────┘
       │
       │ sends message
       ▼
┌──────────────────────────────────────┐
│         LLM Integration              │
│  - System prompts (agent identity)   │
│  - Memory context (if enabled)       │
│  - Tool definitions (beams/prisms)   │
└──────┬───────────────────────────────┘
       │
       │ calls
       ▼
┌──────────────────────────────────────┐
│      LLM Provider (OpenAI, etc.)     │
│  Returns: text | tool_calls | error  │
└──────┬───────────────────────────────┘
       │
       │ response
       ▼
┌──────────────────────────────────────┐
│      Response Processing             │
│  - Execute tool calls                │
│  - Store in memory (if enabled)      │
│  - Return to user                    │
└──────────────────────────────────────┘
```

---

## Trading Workflows

### Pipeline 1: Trade Proposal and Execution

**Purpose:** Market analysis → trade proposal → risk evaluation → conditional execution

**Agents Involved:**
- Market Researcher Agent (proposes trades)
- Risk Manager Agent (evaluates and executes)

**Source Files:**
- `lib/lux/agents/market_researcher.ex`
- `lib/lux/agents/risk_manager.ex`
- `lib/lux/prisms/trade_proposal_prism.ex`
- `lib/lux/beams/hyperliquid/trade_risk_management_beam.ex`

**Complete Pipeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│ STEP 1: Market Analysis (Market Researcher Agent)              │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ User prompt
                            ▼
              ┌──────────────────────────┐
              │  Market Researcher LLM   │
              │  Temperature: 0.7        │
              │  Model: gpt-4o-mini      │
              └──────────┬───────────────┘
                         │
                         │ System Prompt:
                         │ "You are a Market Research Agent specialized in
                         │  analyzing cryptocurrency markets and proposing trades"
                         │
                         │ User Prompt:
                         │ "Based on these market conditions, propose a trade:
                         │  {market_conditions_json}"
                         │
                         ▼
              ┌──────────────────────────┐
              │  JSON Response Schema:   │
              │  {                       │
              │    coin: "ETH",          │
              │    is_buy: true,         │
              │    sz: 0.1,              │
              │    limit_px: 2800.0,     │
              │    order_type: {...},    │
              │    reduce_only: false,   │
              │    rationale: "..."      │
              │  }                       │
              └──────────┬───────────────┘
                         │
                         │ Trade Proposal
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│ STEP 2: Risk Evaluation (Risk Manager Agent)                   │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ TradeProposalSignal sent
                         ▼
              ┌──────────────────────────┐
              │ TradeProposalPrism       │
              │ (Signal Handler)         │
              └──────────┬───────────────┘
                         │
                         │ Sends trade proposal to Risk Manager LLM
                         ▼
              ┌──────────────────────────┐
              │  Risk Manager LLM        │
              │  Temperature: 0.3        │
              │  Model: gpt-4o-mini      │
              └──────────┬───────────────┘
                         │
                         │ System Prompt:
                         │ "You are a Risk Management Agent responsible for
                         │  evaluating trade proposals and executing trades
                         │  that meet risk criteria"
                         │
                         │ Input: Trade proposal + rationale
                         │
                         ▼
              ┌──────────────────────────┐
              │  JSON Response Schema:   │
              │  {                       │
              │    execute_trade: bool,  │
              │    reasoning: "..."      │
              │  }                       │
              └──────────┬───────────────┘
                         │
                         │ Decision
                         ▼
                    ┌────┴────┐
                    │ Branch  │
                    └────┬────┘
                         │
              ┌──────────┴──────────┐
              │                     │
    execute_trade = true   execute_trade = false
              │                     │
              ▼                     ▼
┌──────────────────────────┐    ┌────────────────────────┐
│ STEP 3: Risk Assessment  │    │ Return Rejection       │
│ and Execution            │    │ Signal with reasoning  │
└──────────────────────────┘    └────────────────────────┘
              │
              │ Run Trade Risk Management Beam
              ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Trade Risk Management Beam                     │
│                     (Sequential Execution)                       │
└─────────────────────────────────────────────────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ Parallel Step:      │
    │ Get Portfolio       │
    │ + Market Data       │
    └─────────┬───────────┘
              │
              ├─── HyperliquidUserStatePrism
              │    Input: {address}
              │    Output: {user_state: portfolio_data}
              │
              └─── HyperliquidTokenInfoPrism
                   Input: {}
                   Output: {prices: market_prices}
              │
              │ Both complete, results merged
              ▼
    ┌─────────────────────────────────┐
    │ HyperliquidRiskAssessmentPrism  │
    │ (Uses Python for calculations)  │
    └─────────┬───────────────────────┘
              │
              │ Input: {
              │   portfolio: step1.user_state,
              │   market_data: step2.prices,
              │   proposed_trade: input.trade
              │ }
              │
              ▼
    ┌─────────────────────────────────┐
    │ Risk Metrics Calculated:        │
    │ - position_size_ratio           │
    │ - leverage                      │
    │ - portfolio_concentration       │
    │ - liquidation_risk              │
    │ - unrealized_pnl                │
    └─────────┬───────────────────────┘
              │
              ▼
    ┌─────────────────────┐
    │ Branch Decision:    │
    │ Check Thresholds    │
    └─────────┬───────────┘
              │
              │ Conditions:
              │ - position_size_ratio <= 0.2
              │ - leverage <= 5.0
              │ - portfolio_concentration <= 0.4
              │ - liquidation_risk <= 0.1
              │
              ▼
         ┌────┴────┐
         │ Branch  │
         └────┬────┘
              │
   ┌──────────┴──────────┐
   │                     │
  ALL PASS            ANY FAIL
   │                     │
   ▼                     ▼
┌────────────────┐  ┌────────────────────┐
│ Execute Trade  │  │ Reject Trade       │
│ Prism          │  │ (NoOp Prism)       │
└────┬───────────┘  └────┬───────────────┘
     │                   │
     │ Uses Python       │ Return:
     │ Hyperliquid SDK   │ {
     │                   │   status: "rejected",
     ▼                   │   reason: "...",
┌────────────────┐       │   risk_metrics: {...}
│ Order Result   │       │ }
│ {              │       │
│   status: OK,  │       │
│   order_id,    │       │
│   filled_qty,  │       │
│   ...          │       │
│ }              │       │
└────────────────┘       │
     │                   │
     └───────────┬───────┘
                 │
                 ▼
       ┌─────────────────┐
       │ Return Signal   │
       │ to Risk Manager │
       └─────────────────┘
```

**Data Transformations:**

1. **Market Conditions → Trade Proposal**
   - Input: Market data (prices, volume, trends)
   - Prompt: Market Researcher system prompt + market conditions
   - LLM Output: Structured JSON trade proposal
   - Temperature: 0.7 (balanced analytical creativity)

2. **Trade Proposal → Risk Decision**
   - Input: Trade proposal with rationale
   - Prompt: Risk Manager system prompt + proposal
   - LLM Output: Binary decision (execute_trade: boolean) + reasoning
   - Temperature: 0.3 (conservative decision-making)

3. **Portfolio + Market → Risk Metrics** (Python)
   - Input: Portfolio state, market prices, proposed trade
   - Calculation: Position sizing, leverage, concentration, liquidation risk
   - Output: Numerical risk metrics

4. **Risk Metrics → Execution Decision**
   - Input: Risk metrics from Python calculation
   - Logic: Threshold comparison (hard-coded limits)
   - Output: Branch selection (execute or reject)

5. **Trade Parameters → Order Execution** (Python SDK)
   - Input: Trade details (coin, size, price, order type)
   - API Call: Hyperliquid exchange API
   - Output: Order confirmation or error

**Error Handling:**
- Each beam step supports retries (configurable)
- Fallback mechanisms for recoverable errors
- Detailed logging at each step
- Failed trades return structured error signals

---

