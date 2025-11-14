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

## Chat and Conversation Workflows

### Pipeline 2: Simple Chat Interaction

**Purpose:** User message → agent response with goal-driven prompting

**Source Files:**
- `lib/lux/prisms/handle_chat.ex`
- `lib/lux/agent.ex` (chat function)

**Pipeline Diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    User Sends Chat Message                      │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ message: "Hello, how can you help me?"
                            ▼
              ┌──────────────────────────┐
              │   Chat Signal Created    │
              │   {                      │
              │     message: "...",      │
              │     message_type: "user",│
              │     context: {...}       │
              │   }                      │
              └──────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │  HandleChat Prism        │
              │  (Signal Handler)        │
              └──────────┬───────────────┘
                         │
                         │ Build Dynamic Prompt
                         ▼
              ┌────────────────────────────────────┐
              │ Prompt Template:                   │
              │                                    │
              │ "You are {agent.name}, an AI agent│
              │  with the following goal:         │
              │  {agent.goal}                     │
              │                                    │
              │  You received the following chat  │
              │  message:                         │
              │  {message}                        │
              │                                    │
              │  Generate an appropriate response │
              │  based on your goal and           │
              │  capabilities. Keep the response  │
              │  concise and relevant to the      │
              │  conversation."                   │
              └────────────┬───────────────────────┘
                           │
                           │ Example filled:
                           │ agent.name = "Research Assistant"
                           │ agent.goal = "Help researchers find papers"
                           │ message = "Hello, how can you help me?"
                           ▼
              ┌──────────────────────────┐
              │    LLM Call              │
              │    (Uses agent's config) │
              └──────────┬───────────────┘
                         │
                         │ Response: "I'm a Research Assistant..."
                         ▼
              ┌──────────────────────────┐
              │  Response Signal Created │
              │  {                       │
              │    message: "...",       │
              │    message_type:         │
              │      "response",         │
              │    context: {            │
              │      thread_id,          │
              │      reply_to,           │
              │      metadata            │
              │    }                     │
              │  }                       │
              └──────────┬───────────────┘
                         │
                         ▼
              ┌──────────────────────────┐
              │  Return to User          │
              └──────────────────────────┘
```

**Data Flow:**

```
User Message
     │
     ▼
Chat Signal (Input Schema)
     │
     ├─ message: string
     ├─ message_type: "user" | "system"
     └─ context: {thread_id, metadata}
     │
     ▼
Dynamic Prompt Construction
     │
     ├─ Insert agent.name
     ├─ Insert agent.goal
     └─ Insert user message
     │
     ▼
LLM Processing
     │
     └─ Uses agent.llm_config
        ├─ model
        ├─ temperature
        └─ system messages
     │
     ▼
Chat Signal (Output Schema)
     │
     ├─ message: LLM response
     ├─ message_type: "response"
     └─ context: {thread_id, reply_to, sender_id}
```

---

### Pipeline 3: Chat with Memory

**Purpose:** Conversational agent with context retention across multiple interactions

**Source Files:**
- `lib/lux/agent.ex` (chat function with memory)
- `lib/lux/memory/simple_memory.ex`

**Pipeline Diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│                User Sends Message (use_memory: true)            │
└─────────────────────────────────────────────────────────────────┘
                            │
                            │ message: "My name is Alice"
                            │ opts: [use_memory: true, max_memory_context: 5]
                            ▼
              ┌──────────────────────────┐
              │  Check Memory Enabled?   │
              └──────────┬───────────────┘
                         │
                         │ use_memory == true
                         ▼
              ┌──────────────────────────┐
              │  Retrieve Recent Context │
              │  from Memory             │
              └──────────┬───────────────┘
                         │
                         │ Memory.recent(agent.memory_pid, max_context)
                         ▼
              ┌─────────────────────────────────────┐
              │  Recent Interactions (Last N)       │
              │  [{                                 │
              │    content: "Previous user msg",    │
              │    metadata: {role: :user}          │
              │  }, {                               │
              │    content: "Previous agent resp",  │
              │    metadata: {role: :assistant}     │
              │  }]                                 │
              └──────────┬──────────────────────────┘
                         │
                         │ Convert to LLM message format
                         ▼
              ┌─────────────────────────────────────┐
              │  Messages for LLM:                  │
              │  [                                  │
              │    {role: "system", content: "..."},│
              │    {role: "user", content: "..."},  │
              │    {role: "assistant", "..."},      │
              │    {role: "user", content: "My      │
              │     name is Alice"}  ← NEW          │
              │  ]                                  │
              └──────────┬──────────────────────────┘
                         │
                         │ Enriched context
                         ▼
              ┌──────────────────────────┐
              │    LLM Call with         │
              │    Historical Context    │
              └──────────┬───────────────┘
                         │
                         │ Response: "Nice to meet you, Alice!"
                         ▼
              ┌──────────────────────────┐
              │  Store User Message      │
              │  in Memory               │
              └──────────┬───────────────┘
                         │
                         │ Memory.store(pid, %{
                         │   content: "My name is Alice",
                         │   type: :interaction,
                         │   metadata: %{role: :user}
                         │ })
                         ▼
              ┌──────────────────────────┐
              │  Store Assistant         │
              │  Response in Memory      │
              └──────────┬───────────────┘
                         │
                         │ Memory.store(pid, %{
                         │   content: "Nice to meet you, Alice!",
                         │   type: :interaction,
                         │   metadata: %{role: :assistant}
                         │ })
                         ▼
              ┌──────────────────────────┐
              │  Return Response to User │
              └──────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              Subsequent Message with Memory                     │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ message: "What's my name?"
                         │ opts: [use_memory: true]
                         ▼
              ┌──────────────────────────┐
              │  Retrieve Recent Context │
              └──────────┬───────────────┘
                         │
                         │ Now includes:
                         │ - "My name is Alice" (user)
                         │ - "Nice to meet you..." (assistant)
                         │ - "What's my name?" (user) ← NEW
                         ▼
              ┌──────────────────────────┐
              │    LLM Call with         │
              │    Full Context          │
              └──────────┬───────────────┘
                         │
                         │ Response: "Your name is Alice!"
                         │ (LLM can answer from context)
                         ▼
              ┌──────────────────────────┐
              │  Store Interaction       │
              │  and Return              │
              └──────────────────────────┘
```

**Memory Storage Structure:**

```
ETS Table (SimpleMemory)
Key: {counter, timestamp}
     │
     ▼
Value: {
  id: counter (auto-increment),
  content: any (message text, observation, etc.),
  type: :observation | :reflection | :interaction | :system,
  timestamp: unix_timestamp,
  metadata: %{
    role: :user | :assistant,
    ... custom fields ...
  }
}
```

**Data Transformations:**

1. **User Message → Memory Lookup**
   - Input: Message + use_memory flag
   - Action: Query ETS table for recent N entries
   - Output: List of {content, metadata} tuples

2. **Memory Entries → LLM Messages**
   - Input: Memory entries with role metadata
   - Transform: Convert to OpenAI message format
   - Output: [{role: "user", content: "..."}, ...]

3. **LLM Response → Memory Storage**
   - Input: LLM response text
   - Transform: Wrap in memory structure with timestamp
   - Storage: Insert into ETS with auto-incrementing key

**Memory Retrieval Methods:**

```
┌──────────────────────────────┐
│   Memory Retrieval API       │
└──────────────────────────────┘
         │
         ├─ recent(pid, n)
         │  Returns: Last N entries (reverse chronological)
         │  Use case: Chat context window
         │
         ├─ window(pid, start_time, end_time)
         │  Returns: Entries within time range
         │  Use case: Time-bounded context
         │
         └─ search(pid, query_string)
            Returns: Entries matching text search
            Use case: Semantic retrieval
```

---

