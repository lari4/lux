# Lux AI Prompts Documentation

This document provides a comprehensive catalog of all AI prompts used in the Lux multi-agent system. Each prompt is documented with its purpose, configuration, and exact implementation.

## Table of Contents

1. [Core Agent System Prompts](#core-agent-system-prompts)
   - [Market Researcher Agent](#market-researcher-agent)
   - [Risk Manager Agent](#risk-manager-agent)
2. [Task Processing Prompts](#task-processing-prompts)
3. [Chat and Interaction Prompts](#chat-and-interaction-prompts)
4. [Content Creation Prompts](#content-creation-prompts)
5. [Reflection and Learning Prompts](#reflection-and-learning-prompts)
6. [Documentation and Example Prompts](#documentation-and-example-prompts)

---

## Core Agent System Prompts

These prompts define the fundamental behavior and personality of specialized agents in the Lux system. They establish the agent's role, responsibilities, and output format.

### Market Researcher Agent

**Location:** `lib/lux/agents/market_researcher.ex:76-96`

**Purpose:** Defines a cryptocurrency market analysis agent that proposes trading opportunities based on market conditions. This agent analyzes market data and generates structured trade proposals with complete order specifications.

**LLM Configuration:**
- **Model:** `gpt-4o-mini`
- **Temperature:** `0.7` (balanced creativity for market analysis)
- **Response Format:** Structured JSON with strict schema enforcement
- **Schema:** Trade proposal with fields: coin, is_buy, sz, limit_px, order_type, reduce_only, rationale

**System Prompt:**

```
You are a Market Research Agent specialized in analyzing cryptocurrency markets
and proposing trades. Your recommendations must be in valid JSON format and include
ALL of the following fields:

{
  "coin": "string (e.g., 'ETH', 'BTC')",
  "is_buy": boolean,
  "sz": number,
  "limit_px": number,
  "order_type": {
    "limit": {
      "tif": "Gtc"
    }
  },
  "reduce_only": boolean,
  "rationale": "string explaining the trade"
}

Always include every field with appropriate values based on your analysis.
```

**Dynamic User Prompt Template:**

When proposing a trade, the agent receives market conditions and is prompted with:

```
Based on these market conditions, propose a trade:
{market_conditions_json}

Respond with a complete trade proposal including ALL required fields.
```

**Key Characteristics:**
- Enforces strict JSON schema for trade proposals
- Requires comprehensive field completion (coin, direction, size, price, order type, risk parameters, rationale)
- Temperature set to 0.7 for analytical yet adaptive market recommendations
- Uses GPT-4o-mini for cost-efficient market analysis

---

### Risk Manager Agent

**Location:** `lib/lux/agents/risk_manager.ex:38-50`

**Purpose:** Evaluates trade proposals for risk compliance and makes execution decisions. This agent acts as a gatekeeper, reviewing proposed trades against risk management criteria before allowing execution.

**LLM Configuration:**
- **Model:** `gpt-4o-mini`
- **Temperature:** `0.3` (lower temperature for conservative, consistent risk decisions)
- **Response Format:** Structured JSON with strict schema enforcement
- **Schema:** Risk evaluation with fields: execute_trade (boolean), reasoning (string)
- **Strict Mode:** Enabled (additionalProperties: false) for precise output control

**System Prompt:**

```
You are a Risk Management Agent responsible for evaluating trade proposals
and executing trades that meet risk criteria. You will:

1. Review trade proposals and their rationale
2. Use the Risk Management Beam to evaluate trades
3. Execute approved trades
4. Provide feedback on rejected trades

Respond with a structured evaluation including whether to execute the trade
and your reasoning.
```

**Key Characteristics:**
- Lower temperature (0.3) ensures conservative, consistent risk assessment
- Binary decision output (execute_trade: boolean) with mandatory reasoning
- Designed to work with Risk Management Beam tool for quantitative evaluation
- Strict JSON schema prevents unexpected response variations
- Acts as critical control point in multi-agent trading pipeline

---

## Task Processing Prompts

These prompts are used by the company signal handler to analyze and execute tasks. They enable dynamic task decomposition, tool selection, and execution planning.

### Task Analysis Prompt

**Location:** `lib/lux/agent/companies/signal_handler/default_implementation.ex:214-227`

**Purpose:** Analyzes incoming tasks to determine feasibility, required tools, and execution strategy. This prompt helps the LLM evaluate whether a task can be completed with available resources and what approach should be taken.

**LLM Configuration:**
- **Model:** Inherits from agent/context configuration
- **Temperature:** Inherits from agent/context configuration
- **Response Format:** JSON with AnalysisFeasibilitySchema
- **Schema:** Object with `feasibility.possible` (boolean) and `feasibility.reason` (string)

**Used In:** Task assignment handling within company workflows

**Prompt:**

```
Analyze this task and determine if it is possible to complete with the available tools:
Title: {task_title}
Description: {task_description}

Consider:
1. Is this task possible to complete with the available tools?
2. What type of task is this?
3. What tools might be needed?
4. What should the output look like?
5. Are there any constraints to consider?
6. What artifacts should be produced?
7. Are the requirements clear and complete?
```

**Key Characteristics:**
- Comprehensive task evaluation framework (7 consideration points)
- Feasibility assessment before execution attempts
- Tool availability check against task requirements
- Output format and artifact planning
- Requirement completeness validation
- JSON response enforces structured decision-making
- Prevents wasted resources on impossible tasks

---

### Task Execution Prompt

**Location:** `lib/lux/agent/companies/signal_handler/default_implementation.ex:284-295`

**Purpose:** Guides the LLM to execute a previously analyzed task using available tools. This prompt receives the task analysis results and directs the agent to select appropriate tools, determine parameters, and combine results.

**LLM Configuration:**
- **Model:** Inherits from agent/context configuration
- **Temperature:** Inherits from agent/context configuration
- **Response Format:** Depends on task requirements
- **Tools:** Full access to agent's beams, lenses, and prisms

**Used In:** Task execution phase after successful analysis

**Prompt Template:**

```
Given this task analysis:
{formatted_analysis}

Execute the task using the available tools.
Determine:
1. Which tools to use
2. What parameters to provide
3. How to combine the results

If at any point you determine the task cannot be completed, explain why.
```

**Analysis Format Included:**

The `{formatted_analysis}` contains:
```
Task Type: {task_type}
Required Capabilities: {required_capabilities}
Expected Outputs: {expected_outputs}
Success Criteria: {success_criteria}
```

**Key Characteristics:**
- Two-phase approach: analysis then execution
- Tool selection guided by analysis results
- Parameter determination based on task requirements
- Result combination strategy
- Graceful failure handling with explanations
- Leverages OpenAI's native tool calling for tool execution
- Context-aware execution based on prior analysis

---

## Chat and Interaction Prompts

These prompts handle conversational interactions between agents and enable dynamic response generation based on agent configuration and context.

### Agent Chat Response Prompt

**Location:** `lib/lux/prisms/handle_chat.ex:65-74`

**Purpose:** Generates contextual chat responses based on an agent's identity, goal, and incoming messages. This prompt enables agents to maintain consistent personality and purpose while engaging in conversations.

**LLM Configuration:**
- **Model:** Inherits from agent's llm_config
- **Temperature:** Inherits from agent's llm_config
- **Response Format:** Plain text response
- **Tools:** Currently no tools passed (empty list), but designed to support agent's beams and prisms

**Used In:** HandleChat prism for processing chat signals between agents

**Prompt Template:**

```
You are {agent.name}, an AI agent with the following goal:
{agent.goal}

You received the following chat message:
{message}

Generate an appropriate response based on your goal and capabilities.
Keep the response concise and relevant to the conversation.
```

**Key Characteristics:**
- Dynamic agent identity injection (uses agent.name and agent.goal)
- Goal-driven response generation
- Conciseness emphasis for efficient communication
- Relevance constraint to maintain conversation focus
- Designed for inter-agent communication
- Supports threaded conversations via context metadata

---

