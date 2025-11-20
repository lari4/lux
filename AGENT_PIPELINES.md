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

## Company and Multi-Agent Workflows

### Pipeline 4: Company Objective Execution (CEO-Worker Pattern)

**Purpose:** Hierarchical task delegation from CEO to worker agents based on capabilities

**Source Files:**
- `lib/lux/company/execution_engine/objective_process.ex`
- `lib/lux/agent/companies/signal_handler/default_implementation.ex`
- `lib/lux/schemas/companies/objective_signal.ex`
- `lib/lux/schemas/companies/task_signal.ex`

**Complete Pipeline:**

```
┌─────────────────────────────────────────────────────────────────┐
│          PHASE 1: Objective Initialization                      │
└─────────────────────────────────────────────────────────────────┘

Company.run_objective(company_pid, objective_id, input_data)
                         │
                         ▼
              ┌──────────────────────────┐
              │  Create Objective        │
              │  Process                 │
              │                          │
              │  State: PENDING          │
              │  Steps: [...steps...]    │
              │  Progress: 0%            │
              └──────────┬───────────────┘
                         │
                         │ initialize()
                         ▼
              ┌──────────────────────────┐
              │  State: INITIALIZING     │
              └──────────┬───────────────┘
                         │
                         │ start()
                         ▼
              ┌──────────────────────────┐
              │  State: IN_PROGRESS      │
              └──────────┬───────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│          PHASE 2: CEO Evaluation Loop                           │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ Create ObjectiveSignal
                         ▼
              ┌──────────────────────────────────┐
              │  ObjectiveSignal                 │
              │  Type: "evaluate"                │
              │  Payload: {                      │
              │    objective_id,                 │
              │    title,                        │
              │    steps: [                      │
              │      {index: 0, status:          │
              │       "pending", ...},           │
              │      ...                         │
              │    ],                            │
              │    context: {                    │
              │      available_agents,           │
              │      progress: 0                 │
              │    }                             │
              │  }                               │
              │  Recipient: CEO Agent            │
              └──────────┬───────────────────────┘
                         │
                         │ Routed to CEO
                         ▼
              ┌──────────────────────────┐
              │  CEO Agent               │
              │  handle_objective_       │
              │  evaluation()            │
              └──────────┬───────────────┘
                         │
                         │ Calls evaluate_next_step()
                         ▼
              ┌─────────────────────────────────┐
              │  Determine Next Step            │
              │                                 │
              │  1. Find next pending step      │
              │     (status == "pending")       │
              │                                 │
              │  2. Get step's required         │
              │     capabilities                │
              │                                 │
              │  3. Query AgentHub for agents   │
              │     with matching capabilities  │
              │                                 │
              │  4. Select best available agent │
              └──────────┬──────────────────────┘
                         │
                         │ Example:
                         │ - Step 0: "Gather market data"
                         │ - Required: [:data_collection]
                         │ - Hub returns: [Data Agent]
                         │ - Selected: Data Agent ID
                         ▼
              ┌──────────────────────────────────┐
              │  Create Evaluation Response      │
              │  {                               │
              │    decision: "continue",         │
              │    next_step_index: 0,           │
              │    assigned_agent: "agent_123",  │
              │    required_capabilities:        │
              │      [:data_collection],         │
              │    reasoning: "Selected agent    │
              │      with matching capabilities" │
              │  }                               │
              └──────────┬───────────────────────┘
                         │
                         │ Send back to Company
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│          PHASE 3: Task Assignment and Execution                 │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ Company receives evaluation
                         ▼
              ┌──────────────────────────┐
              │  Create TaskSignal       │
              │  Type: "assignment"      │
              │  Payload: {              │
              │    task_id,              │
              │    objective_id,         │
              │    title: "Gather...",   │
              │    description: "...",   │
              │    required_capabilities,│
              │    context: {            │
              │      tools: [...],       │
              │      constraints,        │
              │      previous_results    │
              │    }                     │
              │  }                       │
              │  Recipient: Data Agent   │
              └──────────┬───────────────┘
                         │
                         │ Routed to assigned agent
                         ▼
              ┌──────────────────────────┐
              │  Data Agent              │
              │  handle_task_            │
              │  assignment()            │
              └──────────┬───────────────┘
                         │
                         │ Check capabilities match
                         ▼
              ┌───────────────────────────┐
              │  Capability Check         │
              │                           │
              │  Required: [:data_...     │
              │            collection]    │
              │  Agent has: [:data_...    │
              │              collection,  │
              │              :analysis]   │
              │                           │
              │  Match? ✓ YES             │
              └──────────┬────────────────┘
                         │
                         ▼
              ┌───────────────────────────┐
              │  Execute Task             │
              │                           │
              │  May involve:             │
              │  - Running prisms/beams   │
              │  - LLM calls              │
              │  - External API calls     │
              └──────────┬────────────────┘
                         │
                         │ Example: Use HyperliquidTokenInfoPrism
                         ▼
              ┌──────────────────────────────┐
              │  Task Result                 │
              │  {                           │
              │    success: true,            │
              │    output: {                 │
              │      prices: {...},          │
              │      volume: {...}           │
              │    },                        │
              │    artifacts: []             │
              │  }                           │
              └──────────┬───────────────────┘
                         │
                         │ Create completion signal
                         ▼
              ┌──────────────────────────┐
              │  TaskSignal              │
              │  Type: "completion"      │
              │  Payload: {              │
              │    task_id,              │
              │    objective_id,         │
              │    status: "completed",  │
              │    result: {...}         │
              │  }                       │
              │  Recipient: CEO Agent    │
              └──────────┬───────────────┘
                         │
                         │ Routed back to CEO
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│          PHASE 4: Progress Update and Iteration                 │
└─────────────────────────────────────────────────────────────────┘
                         │
                         │ CEO receives completion
                         ▼
              ┌──────────────────────────┐
              │  Update Objective        │
              │  Progress                │
              │                          │
              │  - Mark step 0 complete  │
              │  - Store results         │
              │  - Calculate progress    │
              │    (1/3 = 33%)           │
              └──────────┬───────────────┘
                         │
                         │ More steps?
                         ▼
                    ┌────┴────┐
                    │ Branch  │
                    └────┬────┘
                         │
              ┌──────────┴──────────┐
              │                     │
        More steps              All complete
              │                     │
              │                     ▼
              │          ┌──────────────────────────┐
              │          │  ObjectiveSignal         │
              │          │  Type: "completion"      │
              │          │  Payload: {              │
              │          │    objective_id,         │
              │          │    status: "completed",  │
              │          │    progress: 100,        │
              │          │    final_result: {...},  │
              │          │    artifacts: [...]      │
              │          │  }                       │
              │          └──────────────────────────┘
              │                     │
              │                     ▼
              │          ┌──────────────────────────┐
              │          │  Company State:          │
              │          │  COMPLETED               │
              │          └──────────────────────────┘
              │
              │ Loop back to PHASE 2 (evaluate next step)
              ▼
      (Repeat for steps 1, 2, etc.)
```

**Data Flow Through Prompts:**

```
Step 0: CEO Evaluation
  ↓
  Prompt: (No explicit LLM prompt in default implementation)
  Logic:  Pure Elixir - find next pending step, match capabilities
  Output: {decision, next_step_index, assigned_agent, reasoning}
  ↓
Step 1: Task Assignment
  ↓
  Signal: TaskSignal with context
  Input:  {task description, required capabilities, context}
  ↓
Step 2: Agent Execution
  ↓
  Prompt: (Agent-specific, may use Task Analysis/Execution prompts)

  Task Analysis Prompt:
  "Analyze this task and determine if it is possible to complete
   with the available tools:
   Title: {task_title}
   Description: {task_description}

   Consider:
   1. Is this task possible to complete with the available tools?
   2. What type of task is this?
   3. What tools might be needed?
   4. What should the output look like?
   5. Are there any constraints to consider?
   6. What artifacts should be produced?
   7. Are the requirements clear and complete?"

  Output: {feasibility: {possible: true/false, reason: "..."}}
  ↓

  If feasible, Task Execution Prompt:
  "Given this task analysis:
   {formatted_analysis}

   Execute the task using the available tools.
   Determine:
   1. Which tools to use
   2. What parameters to provide
   3. How to combine the results

   If at any point you determine the task cannot be completed,
   explain why."

  Output: Task result (success/failure with data)
  ↓
Step 3: Result Storage
  ↓
  TaskSignal(type: "completion")
  Result stored in objective context
  Available to subsequent steps
  ↓
Step 4: Progress Calculation
  ↓
  completed_steps / total_steps * 100
  Update objective state
```

**Agent Collaboration Pattern:**

```
         AgentHub
             │
             │ Provides agent registry
             ▼
         CEO Agent
             │
             ├─ Queries capabilities
             ├─ Selects agents
             └─ Assigns tasks
             │
             ▼
    ┌────────┴─────────┬──────────┐
    │                  │          │
Data Agent      Analysis Agent  Writer Agent
[:data_...]     [:analysis]     [:writing,
 collection]                     :editing]
    │                  │          │
    └──────────────────┴──────────┘
             │
             │ All report back to CEO
             ▼
         CEO Agent
             │
             └─ Assembles final result
```

---

### Pipeline 5: Content Creation Multi-Agent Workflow

**Purpose:** Research → Outline → Draft → Edit pipeline with specialized agents

**Source Files:**
- `test/support/agents/writer.ex`
- `guides/multi_agent_collaboration.livemd`

**Pipeline Diagram:**

```
┌─────────────────────────────────────────────────────────────────┐
│                STEP 1: Research Phase                           │
└─────────────────────────────────────────────────────────────────┘

User Request: "Create blog post about benefits of exercise"
              │
              ▼
     ┌────────────────────┐
     │  Research Agent    │
     │  Capabilities:     │
     │  [:research,       │
     │   :analysis]       │
     └─────────┬──────────┘
               │
               │ System Prompt:
               │ "You are a Research Assistant specialized in
               │  finding and analyzing information.
               │  Work with other agents to provide comprehensive
               │  research results."
               │
               │ User Prompt:
               │ "Research the benefits of exercise"
               ▼
     ┌────────────────────┐
     │  LLM Research      │
     │  Model: gpt-4o-mini│
     │  Max tokens: 500   │
     └─────────┬──────────┘
               │
               │ Research findings
               ▼
     ┌────────────────────────────┐
     │  Research Result           │
     │  {                         │
     │    topic: "Exercise",      │
     │    key_findings: [         │
     │      "Improves cardio...", │
     │      "Reduces stress...",  │
     │      "Enhances mood..."    │
     │    ],                      │
     │    sources: [...]          │
     │  }                         │
     └─────────┬──────────────────┘
               │
               │ Pass to next agent
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                STEP 2: Outline Creation                         │
└─────────────────────────────────────────────────────────────────┘
               │
               ▼
     ┌────────────────────┐
     │  Writer Agent      │
     │  Task: create_     │
     │        outline     │
     └─────────┬──────────┘
               │
               │ Prompt:
               │ "Create a detailed outline based on this research:
               │  {research}
               │
               │  Include:
               │  1. Introduction section
               │  2. Main sections with key points
               │  3. Supporting subsections
               │  4. Conclusion section
               │
               │  Respond with a JSON object containing:
               │  {
               │    title: 'proposed title',
               │    sections: [
               │      {title, points, subsections: [{...}]}
               │    ]
               │  }"
               ▼
     ┌────────────────────┐
     │  LLM Processing    │
     │  Temperature: 0.7  │
     └─────────┬──────────┘
               │
               │ Structured outline
               ▼
     ┌──────────────────────────────────┐
     │  Outline Result                  │
     │  {                               │
     │    title: "10 Amazing Benefits   │
     │             of Regular Exercise",│
     │    sections: [                   │
     │      {                           │
     │        title: "Physical Health", │
     │        points: ["Cardio", ...],  │
     │        subsections: [...]        │
     │      },                          │
     │      {                           │
     │        title: "Mental Health",   │
     │        ...                       │
     │      }                           │
     │    ]                             │
     │  }                               │
     └─────────┬────────────────────────┘
               │
               │ Pass to draft generation
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                STEP 3: Draft Writing                            │
└─────────────────────────────────────────────────────────────────┘
               │
               ▼
     ┌────────────────────┐
     │  Writer Agent      │
     │  Task: write_draft │
     └─────────┬──────────┘
               │
               │ Prompt:
               │ "Write a blog post draft following this outline:
               │  {outline}
               │
               │  Ensure:
               │  1. Engaging introduction
               │  2. Clear flow between sections
               │  3. Supporting evidence for claims
               │  4. Strong conclusion
               │  5. Appropriate tone and style
               │
               │  Respond with a JSON object containing:
               │  {
               │    title: 'final title',
               │    content: 'full blog post content',
               │    metadata: {
               │      word_count: number,
               │      reading_time: '5 min',
               │      target_audience: '...'
               │    }
               │  }"
               ▼
     ┌────────────────────┐
     │  LLM Processing    │
     │  Temperature: 0.7  │
     └─────────┬──────────┘
               │
               │ Complete draft
               ▼
     ┌──────────────────────────────────┐
     │  Draft Result                    │
     │  {                               │
     │    title: "10 Amazing Benefits   │
     │            of Regular Exercise", │
     │    content: "Exercise is one of  │
     │              the most powerful...",
     │    metadata: {                   │
     │      word_count: 1200,           │
     │      reading_time: "6 min",      │
     │      target_audience: "Adults    │
     │        interested in fitness"    │
     │    }                             │
     │  }                               │
     └─────────┬────────────────────────┘
               │
               │ Pass to editing
               ▼
┌─────────────────────────────────────────────────────────────────┐
│                STEP 4: Content Editing                          │
└─────────────────────────────────────────────────────────────────┘
               │
               ▼
     ┌────────────────────┐
     │  Writer Agent      │
     │  Task: edit_content│
     └─────────┬──────────┘
               │
               │ Prompt:
               │ "Edit and improve this content:
               │  {content}
               │
               │  Focus on:
               │  1. Clarity and conciseness
               │  2. Grammar and style
               │  3. Flow and transitions
               │  4. Technical accuracy
               │  5. Engagement factor
               │
               │  Respond with a JSON object containing:
               │  {
               │    edited_content: 'improved content',
               │    changes_made: ['list', 'of', 'changes'],
               │    improvement_metrics: {
               │      clarity: 'score 1-10',
               │      engagement: 'score 1-10',
               │      technical_accuracy: 'score 1-10'
               │    }
               │  }"
               ▼
     ┌────────────────────┐
     │  LLM Processing    │
     │  Temperature: 0.7  │
     └─────────┬──────────┘
               │
               │ Edited content with metrics
               ▼
     ┌──────────────────────────────────┐
     │  Final Result                    │
     │  {                               │
     │    edited_content: "...",        │
     │    changes_made: [               │
     │      "Improved intro clarity",   │
     │      "Added transition phrases", │
     │      "Fixed grammar in para 3"   │
     │    ],                            │
     │    improvement_metrics: {        │
     │      clarity: "9",               │
     │      engagement: "8",             │
     │      technical_accuracy: "10"    │
     │    }                             │
     │  }                               │
     └──────────────────────────────────┘
```

**Prompt Chain Summary:**

```
Research Prompt (Agent 1)
  ↓ research findings
Outline Prompt (Agent 2)
  ↓ structured outline
Draft Prompt (Agent 2)
  ↓ complete draft
Edit Prompt (Agent 2)
  ↓ final polished content
```

**Data Transformations:**

1. **User Request → Research Findings**
   - Prompt: Research agent system prompt + topic
   - Output: Key findings, sources, summary

2. **Research → Outline**
   - Prompt: Outline creation prompt + research data
   - Output: Hierarchical structure (title, sections, points, subsections)

3. **Outline → Draft**
   - Prompt: Draft writing prompt + outline
   - Output: Full content + metadata (word count, reading time, audience)

4. **Draft → Edited Content**
   - Prompt: Editing prompt + draft
   - Output: Improved content + change log + quality metrics

---

## Signal-Based Workflows

### Signal Architecture and Routing

**Source Files:**
- `lib/lux/signal.ex`
- `lib/lux/signal/router.ex`
- `lib/lux/agent.ex` (handle_info callbacks)

**Complete Signal Flow:**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Signal Creation                              │
└─────────────────────────────────────────────────────────────────┘

Agent A creates signal
                │
                ▼
     ┌────────────────────────────────┐
     │  %Lux.Signal{                  │
     │    id: UUID,                   │
     │    schema_id: TaskSignal,      │
     │    payload: %{                 │
     │      type: "assignment",       │
     │      task_id: "...",           │
     │      description: "...",       │
     │      context: {...}            │
     │    },                          │
     │    sender: agent_a_id,         │
     │    recipient: agent_b_id,      │
     │    timestamp: DateTime,        │
     │    metadata: %{...}            │
     │  }                             │
     └────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Signal Routing                               │
└─────────────────────────────────────────────────────────────────┘
                  │
                  │ Signal.Router.route(signal, router: pid, hub: hub_pid)
                  ▼
     ┌────────────────────────────────┐
     │  Signal Router Process         │
     │                                │
     │  1. Extract recipient ID       │
     │  2. Query AgentHub for         │
     │     recipient's PID            │
     │  3. Verify recipient exists    │
     └────────────┬───────────────────┘
                  │
                  ▼
        ┌─────────┴──────────┐
        │                    │
   Recipient Found     Recipient Not Found
        │                    │
        ▼                    ▼
     Send message      Return error
     {:signal, sig}    {:error, :not_found}
        │
        │
        ▼
┌─────────────────────────────────────────────────────────────────┐
│                  Signal Reception (Agent B)                     │
└─────────────────────────────────────────────────────────────────┘
        │
        │ Agent B receives {:signal, signal}
        ▼
     ┌────────────────────────────────┐
     │  Agent.handle_info/2           │
     │  ({:signal, signal}, state)    │
     └────────────┬───────────────────┘
                  │
                  │ Look up signal handler
                  ▼
     ┌────────────────────────────────┐
     │  Find Handler for Schema       │
     │                                │
     │  agent.signal_handlers         │
     │  |> Enum.find(fn {schema, _}  │
     │       schema == signal.        │
     │                 schema_id      │
     │     end)                       │
     └────────────┬───────────────────┘
                  │
                  ▼
        ┌─────────┴──────────┐
        │                    │
   Handler Found         No Handler
        │                    │
        ▼                    ▼
┌──────────────────┐   ┌───────────────┐
│  Determine Type  │   │  Log warning  │
└────────┬─────────┘   │  Ignore signal│
         │             └───────────────┘
         │
         ▼
  ┌──────┴──────┬──────────┬──────────┐
  │             │          │          │
Prism        Beam       Lens    Module/Function
  │             │          │          │
  ▼             ▼          ▼          ▼
Execute     Execute    Execute   Call function
with        workflow   data      with signal
input       steps      fetch     and context
  │             │          │          │
  └─────────────┴──────────┴──────────┘
                  │
                  │ Handler execution result
                  ▼
     ┌────────────────────────────────┐
     │  Process Handler Result        │
     │                                │
     │  {:ok, response_signal}        │
     │  or                            │
     │  {:error, reason}              │
     └────────────┬───────────────────┘
                  │
                  ▼
        ┌─────────┴──────────┐
        │                    │
    Success              Error
        │                    │
        ▼                    ▼
┌──────────────────┐   ┌───────────────┐
│  Route Response  │   │  Log error    │
│  Signal back to  │   │  May send     │
│  original sender │   │  error signal │
└──────────────────┘   └───────────────┘
```

**Signal Handler Types:**

```
Signal Handler Configuration:
  signal_handlers: [
    {SchemaModule, HandlerSpec}
  ]

Handler Spec Types:

1. Prism Handler:
   {TaskSignal, MyPrism}
   → Executes: MyPrism.handler(signal.payload, agent_context)

2. Beam Handler:
   {ObjectiveSignal, MyBeam}
   → Executes: MyBeam.run(signal.payload, agent_context)

3. Lens Handler:
   {DataRequestSignal, MyLens}
   → Executes: MyLens.fetch(signal.payload, agent_context)

4. Module/Function Handler:
   {TaskSignal, {MyModule, :my_function}}
   → Executes: MyModule.my_function(signal, agent_context)
```

**Signal Schemas:**

```
1. TaskSignal
   Payload Types:
   - "assignment" → New task for agent
   - "status_update" → Progress update
   - "completion" → Task finished successfully
   - "failure" → Task failed with error

2. ObjectiveSignal
   Payload Types:
   - "evaluate" → CEO evaluates next step
   - "next_step" → CEO assigns next step
   - "status_update" → Progress update
   - "completion" → Objective completed

3. ChatSignal
   Payload Types:
   - "user" → Message from user
   - "response" → Agent response
   - "error" → Error message

4. TradeProposalSignal
   Payload Types:
   - Trade proposal data
   → Handled by TradeProposalPrism
```

---

## Summary and Key Patterns

### Common Pipeline Patterns

#### 1. Linear Sequential Pipeline

```
Input
  ↓
[Step 1: Process A] → Prompt A → LLM → Result A
  ↓
[Step 2: Process B] → Prompt B (uses Result A) → LLM → Result B
  ↓
[Step 3: Process C] → Prompt C (uses Result B) → LLM → Final Output
```

**Examples:**
- Content creation (Research → Outline → Draft → Edit)
- Each step uses output from previous step
- Linear data dependency

#### 2. Parallel with Merge Pipeline

```
Input
  ↓
  ├─ [Parallel Step 1] → Result 1
  ├─ [Parallel Step 2] → Result 2
  └─ [Parallel Step 3] → Result 3
  ↓
[Merge/Combine] → Uses all parallel results
  ↓
Final Output
```

**Examples:**
- Trading workflow (Portfolio State + Market Data in parallel)
- Independent data gathering operations
- Results combined for decision-making

#### 3. Conditional Branching Pipeline

```
Input
  ↓
[Evaluation Step] → Decision
  ↓
  ├─ Condition A → [Branch A Steps] → Output A
  ├─ Condition B → [Branch B Steps] → Output B
  └─ Default → [Default Steps] → Output Default
```

**Examples:**
- Risk management (approve/reject based on thresholds)
- Task routing (based on capabilities)
- Error handling (retry/fallback/fail)

#### 4. Event-Driven Multi-Agent Pipeline

```
Agent A
  ↓
  Creates Signal
  ↓
  Router
  ↓
Agent B
  ↓
  Processes
  ↓
  Creates Response Signal
  ↓
  Router
  ↓
Agent A
```

**Examples:**
- Company objective execution (CEO → Workers → CEO)
- Agent collaboration (Research → Writing)
- Asynchronous task distribution

#### 5. Memory-Enhanced Conversational Pipeline

```
User Message
  ↓
Retrieve Memory Context (Last N interactions)
  ↓
Enrich Prompt with Historical Context
  ↓
LLM Processing
  ↓
Store User Message + Response in Memory
  ↓
Return Response
```

**Examples:**
- Chat with memory
- Context retention across conversations
- Learning from past interactions

---

### Prompt Chaining Patterns

#### 1. Analysis → Execution Chain

```
Task Input
  ↓
Analysis Prompt:
  "Analyze this task and determine if it is possible..."
  ↓
Feasibility Result: {possible: true/false, reason}
  ↓
IF possible:
  Execution Prompt:
    "Given this task analysis: {analysis}
     Execute the task using available tools..."
  ↓
  Execution Result
```

**Used in:** Task processing, company workflows

#### 2. Proposal → Evaluation → Action Chain

```
Market Data
  ↓
Proposal Prompt (Market Researcher):
  "Analyze cryptocurrency markets and propose trades..."
  ↓
Trade Proposal: {coin, is_buy, sz, limit_px, rationale}
  ↓
Evaluation Prompt (Risk Manager):
  "Evaluate trade proposals and executing trades that
   meet risk criteria..."
  ↓
Decision: {execute_trade: boolean, reasoning}
  ↓
IF execute_trade:
  Risk Assessment Beam (No prompt, Python calculations)
  ↓
  Execution
```

**Used in:** Trading workflows

#### 3. Progressive Refinement Chain

```
Raw Input (Research)
  ↓
Structure Prompt:
  "Create a detailed outline based on this research..."
  ↓
Outline
  ↓
Expansion Prompt:
  "Write a blog post draft following this outline..."
  ↓
Draft
  ↓
Refinement Prompt:
  "Edit and improve this content..."
  ↓
Final Content
```

**Used in:** Content creation workflows

---

### Data Transformation Patterns

#### 1. Enrichment Pattern

```
Basic Data → Context Addition → Enriched Data

Example:
User Message
  + Agent Identity (name, goal)
  + Memory Context (past interactions)
  + Available Tools (beams, prisms)
  ↓
Enriched Prompt for LLM
```

#### 2. Aggregation Pattern

```
Multiple Sources → Combine → Single Result

Example:
Portfolio State + Market Prices + Proposed Trade
  ↓
Risk Calculation (Python)
  ↓
Unified Risk Metrics
```

#### 3. Transformation Pattern

```
Input Format A → Conversion → Output Format B

Example:
Memory Entries (ETS table)
  ↓
Convert to LLM message format
  ↓
[{role: "user", content: "..."}, ...]
```

#### 4. Filtering Pattern

```
Large Dataset → Filter Criteria → Subset

Example:
All Agents in Hub
  ↓
Filter by capability: :research
  ↓
Research Agents Only
```

---

### Integration Points Summary

#### LLM Integration

```
Agent Configuration
  ↓
LLM Config (model, temperature, messages)
  + Tool Definitions (beams, prisms, lenses)
  + Memory Context (if enabled)
  ↓
LLM.call(prompt, tools, config)
  ↓
OpenAI API
  ↓
Response (text | tool_calls | error)
  ↓
Process response
  ├─ Execute tool calls
  ├─ Store in memory
  └─ Return to user/agent
```

#### Python Integration

```
Elixir Agent
  ↓
Prism with ~PY"""..."""
  ↓
Lux.Python module
  ↓
Venomous library (Python bridge)
  ↓
Python execution
  ├─ NumPy calculations
  ├─ Hyperliquid SDK
  └─ ML models
  ↓
Return to Elixir as native types
```

#### Blockchain Integration

```
Agent/Prism
  ↓
Ethers library
  ↓
Ethereum RPC calls
  ├─ Contract interactions
  ├─ Balance queries
  ├─ Transaction submission
  └─ Event listening
  ↓
Return blockchain data
```

#### Memory Integration

```
Agent with memory_config
  ↓
SimpleMemory backend (ETS)
  ↓
Operations:
  ├─ store(entry)
  ├─ recent(n)
  ├─ window(start, end)
  └─ search(query)
  ↓
Used in chat for context
```

---

### Error Handling Patterns

#### 1. Retry with Backoff

```
Step Execution
  ↓
Error
  ↓
Retries remaining?
  YES → Wait (backoff) → Retry
  NO → Check fallback
```

#### 2. Fallback Pattern

```
Primary execution
  ↓
Failure
  ↓
Fallback defined?
  YES → Execute fallback → Continue or Stop
  NO → Propagate error
```

#### 3. Graceful Degradation

```
Full feature execution
  ↓
Error
  ↓
Return partial result with error flag
  {status: "partial", data: {...}, error: "..."}
```

---

### Performance Considerations

#### Parallel Execution

- Use parallel beam steps for independent operations
- Example: Fetch portfolio and market data simultaneously
- Reduces total pipeline time

#### Memory Context Limits

- Control `max_memory_context` to limit token usage
- Default: 5 recent interactions
- Balance between context and cost

#### Model Selection

- Use `gpt-4o-mini` for cost-efficient operations
- Reserve `gpt-4` for complex reasoning (reflection, strategy)
- Temperature tuning: 0.0-0.3 (conservative) vs 0.7+ (creative)

#### Tool Call Optimization

- Minimize tool definitions passed to LLM (only relevant tools)
- Use structured outputs (JSON schemas) for predictable parsing
- Batch operations where possible

---

### Testing Patterns

#### Integration Testing

```elixir
# Real OpenAI API calls
# Deterministic responses: temperature=0.0, seed=42
# Verify persona consistency across chats
# Test memory retention and recall
```

#### Unit Testing

```elixir
# Mock LLM responses
# Test signal routing
# Verify capability matching
# Test beam step orchestration
```

---

## Workflow Summary Table

| Workflow | Prompts Used | Data Flow | Agents/Tools | Key Features |
|----------|-------------|-----------|--------------|--------------|
| **Trading** | Market Researcher → Risk Manager | Market data → Proposal → Risk eval → Execution | Market Researcher, Risk Manager, Hyperliquid prisms/beams | Parallel data gathering, Python risk calc, conditional execution |
| **Simple Chat** | Chat response prompt | User msg → LLM → Response | Any agent with HandleChat prism | Dynamic agent identity injection, goal-driven responses |
| **Chat with Memory** | Chat response prompt + memory context | User msg → Memory lookup → Enriched prompt → LLM → Store & respond | Any agent with memory_config | Context retention, ETS storage, configurable window |
| **Company Objective** | Task analysis → Task execution | Objective → CEO evaluation → Task assignment → Execution → Results | CEO agent, worker agents, AgentHub | Hierarchical delegation, capability-based routing, progress tracking |
| **Content Creation** | Research → Outline → Draft → Edit | User request → Research → Structured outline → Full draft → Polished content | Research agent, Writer agent | Sequential refinement, progressive elaboration, quality metrics |
| **Signal Routing** | N/A (signal handlers) | Signal creation → Router → Recipient → Handler execution → Response | Signal Router, AgentHub, all agents | Event-driven, async communication, typed signals |

---

## Conclusion

The Lux agent system provides a sophisticated framework for building multi-agent AI applications with:

- **Modular Tools**: Lenses (data), Prisms (actions), Beams (workflows)
- **Flexible Orchestration**: Sequential, parallel, conditional execution
- **Event-Driven Communication**: Signal-based routing with typed schemas
- **Memory Integration**: Context retention across conversations
- **Multi-Agent Collaboration**: Capability-based discovery and delegation
- **Robust Error Handling**: Retries, fallbacks, graceful degradation
- **External Integrations**: LLMs, Python, Blockchain, APIs

Each pipeline demonstrates different patterns of prompt chaining, data transformation, and agent coordination, providing a comprehensive toolkit for building complex agentic systems.

